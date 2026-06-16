# 1. 组合（Composition）

> Shell 不面向对象，不面向函数，它面向进程。

---

## 管道不是语法糖——是 OS 级别的函数式编程

`|` 不是语法糖，它做的事情其他语言学不来：**创建 pipe，fork 子进程，各自 exec，一个写 fd 一个读 fd，内核调度**。

```bash
# 组合三个进程，各自独立，内核调度
cat access.log | grep '500' | awk '{print $1}' | sort -u
```

每个命令是一个独立进程，不是函数调用。这意味着：
- 单进程崩溃不影响管道整体退出（`pipefail` 可以解决，但默认不传透）
- 可以用 `timeout` 给任意一段插入超时控制
- 可以随时替换其中任意一个环节为远程命令

```bash
# 把中间环节换成远程执行，管道结构不变
cat access.log | ssh logsrv 'grep "500"' | awk '{print $1}' | sort -u
```

---

## `<( )` 进程替换：把远程主机当文件用

管道解决不了"这个命令不接受 stdin，只要文件名参数"的问题。`<( )` 就是补这个缺的。

```bash
# 对比两台机器的 /etc/hosts
diff <(ssh host1 cat /etc/hosts) <(ssh host2 cat /etc/hosts)

# 对比本地和远程的 nginx 配置
diff /etc/nginx/nginx.conf <(ssh web cat /etc/nginx/nginx.conf)

# 把命令输出传给只认文件参数的工具
vimdiff <(curl -s https://api.example.com/v1/config) local.conf
```

**原理**：`<(cmd)` 创建一个 named pipe（`/dev/fd/N`），Shell 把 `cmd` 的 stdout 重定向到这个 fd，然后把路径作为参数传给外层命令。

---

## `|&` 同时转发 stdout + stderr

`2>&1 |` 写多了烦，`|&` 是它的简写。

```bash
# 把 stdout 和 stderr 一起送去 grep
some_noisy_command |& grep 'ERROR'

# 等价于
some_noisy_command 2>&1 | grep 'ERROR'
```

当你写个包装脚本需要**原样透传**子进程的全部输出时有用：

```bash
wrapper() {
    "$@" |& tee -a "$LOGFILE"
}
```

---

## 管道的 SIGPIPE：区分"被掐断"和"真报错"

这是管道的经典暗坑。默认行为：管道读端关闭时，写端收到 SIGPIPE（信号 13）。进程默认退出码 **141**（128 + 13）。

```bash
# head 读完前 5 行就关闭 stdin，上游 awk 收到 SIGPIPE
seq 1000000 | head -5
echo $?  # 可能是 0（head 成功），也可能是 141（awk 被 SIGPIPE 杀死）
```

有了 `set -o pipefail` 之后，这个 141 会传递出去：

```bash
set -o pipefail
seq 1000000 | head -5
echo $?  # 141 — 这其实是正常行为，但你的脚本以为它错了
```

**处理方式**：

```bash
# 允许 141 通过（SIGPIPE 不算故障）
some_pipeline() {
    set -o pipefail
    "$@" || [[ $? -eq 141 ]]  # SIGPIPE 不算错误
}
```

---

## `&` + `wait` + `$!`：并行编排通用模板

同时操作 N 台机器，等全部结束后汇总结果：

```bash
# 并行 ssh 执行
HOSTS=(web-01 web-02 web-03 db-01)
PIDS=()
FAILED=0

for host in "${HOSTS[@]}"; do
    (
        ssh "$host" "systemctl reload nginx"
        exit $?
    ) &
    PIDS+=($!)
done

# 等待所有后台任务完成
for pid in "${PIDS[@]}"; do
    wait "$pid" || ((FAILED++))
done

echo "[INFO] $FAILED hosts failed"
exit "$FAILED"
```

关键点：
- `$!` 必须紧跟在 `&` 之后立即捕获，否则被其他命令覆盖
- `wait $pid` 获取子进程退出码
- 子进程用 `( )` 包裹，避免变量污染与信号隔离

---

## 管道与 `set -o pipefail` 的微妙关系

标配 `set -euo pipefail` 在管道场景会引入隐藏风险：

```bash
set -euo pipefail

# 如果 head 读完关闭管道，grep 被 SIGPIPE 杀死 → 整个脚本退出
grep 'ERROR' huge.log | head -5
```

**问题**：`grep` 的 141（SIGPIPE）被 `pipefail` 传播，然后 `-e` 捕获到非零退出码，脚本直接崩了。

**推荐的写法不是一刀切**：

```bash
# 更精细的做法
set -Euo pipefail

# 对明确知道可能被 SIGPIPE 的管道，局部关闭 -e
grep_safe() {
    set +e
    grep "$@" | head -5
    local rc=$?
    set -e
    # SIGPIPE(141) 不算错误
    return $(( rc == 141 ? 0 : rc ))
}
```

或者更极简的做法——很多老运维只用 `set -u`，手工处理关键错误，反而更可控。

---

## 设计可管道的脚本——让你的程序也能被 `|`

前面都在讲怎么消费管道。这一节讲怎么**生产**——让自己写的脚本成为管道中合格的一环。

### stdin/stdout 是默认接口

```bash
# 坏的：只接受文件名参数，和管道不兼容
parse_logs() {
    local file="$1"
    grep 'ERROR' "$file" | awk '{print $1, $2}'
}
# 你想这样用不行了：tail -f app.log | parse_logs

# 好的：没文件就从 /dev/stdin 读
parse_logs() {
    local input="${1:-/dev/stdin}"
    grep 'ERROR' "$input" | awk '{print $1, $2}'
}
# 两种用法都支持：
# parse_logs /var/log/app.log
# tail -f /var/log/app.log | parse_logs
```

支持 `-` 作为标准占位符：

```bash
parse_logs() {
    local input="$1"
    [[ "$input" == "-" ]] && input="/dev/stdin"
    ...
}
```

### 错误/日志走 stderr，status 走 stdout

这是最核心的管道契约。stdout 归下游，stderr 归终端。

```bash
# 坏的：和 grep 不兼容
check_health() {
    if curl -sf http://localhost/health; then
        echo "OK"
    else
        echo "FAIL"
    fi
}

check_health | grep "ERROR"
# 永远匹配不到——"OK" 和 "FAIL" 都混在 stdout 里

# 好的：状态信息走 stderr，结果走 stdout
check_health() {
    if curl -sf http://localhost/health; then
        echo "healthy"     # stdout：给下游
        echo "[OK] health check passed" >&2  # stderr：给人看
    else
        echo "unhealthy"
        echo "[ERROR] health check failed" >&2
    fi
}

check_health | grep "unhealthy"  # 正常工作了
```

### 流式处理，不加载全部到内存

```bash
# 坏的：处理过程中加载全部
process_lines() {
    local lines
    lines=$(cat)          # 全部读入内存
    while IFS= read -r line; do
        process "$line"
    done <<< "$lines"
}

# 好的：边读边吐，10MB 和 10GB 一样处理
process_lines() {
    while IFS= read -r line; do
        process "$line"
    done
}
```

### 退出码要有意义

下游靠 `$?` 判断成败。不要所有情况都 `exit 0`。

```bash
# 坏的：下游永远是 0
validate_config() {
    if [[ -f "$1" ]]; then
        echo "valid"
    else
        echo "invalid"
    fi
    exit 0  # grep "invalid" && ... 永远不会触发
}

# 好的：0 = 正常，非 0 = 异常
validate_config() {
    if [[ -f "$1" ]]; then
        echo "valid"
        return 0
    else
        echo "invalid" >&2
        return 1
    fi
}
```

退出码惯例：**0 = 成功，1 = 通用错误，2 = 用法错误**，其他码留给具体语义。

### 优雅处理 SIGPIPE

下游 `head` / `awk` 读完就关管道，你还在写 stdout 就会收到 SIGPIPE。

```bash
# 不优雅：收到 SIGPIPE 还傻愣着
produce_data() {
    for i in $(seq 1000000); do
        echo "line $i"
    done
}
produce_data | head -5
# seq 被 SIGPIPE 杀掉，退出码 141

# 优雅：写 stdout 时容忍 SIGPIPE
produce_data() {
    for i in $(seq 1000000); do
        echo "line $i" || break   # 写失败就停（管道断了）
    done
}
produce_data | head -5
# 管道断开后 echo 失败 → break → 正常退出 0
```
