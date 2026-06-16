# 2. 防御性编程

> 不是写得越多越安全。简洁本身就是可维护性。

---

## 单实例锁（flock）的正确写法

最常见的错误是用 pidfile + `kill -0` 检测——进程 ID 可能被复用，pidfile 可能残留。

**正确的做法**：用 `flock` 操作一个文件描述符，文件锁随 fd 自动释放。

```bash
LOCKFILE="/var/run/myscript.lock"

exec 200>"$LOCKFILE"                # 打开/创建锁文件，绑定 fd 200
flock -n 200 || {                   # -n 非阻塞
    echo "Another instance is running" >&2
    exit 1
}

# 脚本退出时自动释放锁（fd 关闭 → 内核释放锁）
```

注意点：
- 锁文件用 `>` 而非 `<>`：用 `>` 会清空重建，刷新锁状态
- fd 编号随意，避免与 stdin/stdout/stderr 冲突，一般 200+
- `flock -n` 失败时锁不占用，不残留

**pidfile 的问题**：

```bash
# 不要这么写
echo $$ > /var/run/script.pid
# 进程挂了，pid 被复用 → 另一个进程拿到同一个 pid → 以为在运行
# 脚本异常退出没 cleanup → pidfile 成僵尸
```

---

## 管道错误为什么被吞掉 & 怎么不漏

```bash
# 你以为所有命令都成功了，但 grep 第二行就报错了
cat data.csv | grep '^[0-9]' | awk -F, '{print $2}' > output.txt
# $? 是 awk 的退出码，grep 中间报错你不知道
```

默认情况下，管道返回**最后一个命令**的退出码。前面的命令失败被静默吞噬。

**解法：`set -o pipefail`**：

```bash
set -o pipefail
cat data.csv | grep '^[0-9]' | awk -F, '{print $2}' > output.txt
# 现在 $? 反映管道中所有命令的成败
```

注意 SIGPIPE 的误报——参考第一章的 `$? -eq 141` 处理。

---

## 临时文件的安全创建与退出清理

```bash
# 错误示范
TMPFILE="/tmp/myscript-$$.tmp"
# 可预测路径，其他进程可以猜到，/tmp 下的竞态问题
# 脚本崩溃时没人清理
```

**正确做法**：

```bash
TMPDIR=$(mktemp -d) || { echo "mktemp failed" >&2; exit 1; }
trap 'rm -rf -- "$TMPDIR"' EXIT   # 确保任何退出路径都清理

TMPFILE="$TMPDIR/data.tmp"
# ... 使用临时文件 ...
```

`EXIT` 会覆盖 `exit`、`SIGINT`、`SIGTERM`、正常执行结束，但**不**覆盖 `SIGKILL`（谁也救不了）。

如果脚本被 `kill -9`，临时文件会残留。可以加个 systemd-tmpfiles 兜底清理，不过生产上常见的选择是 **用 `flock` + 固定路径覆盖写**，省去维护临时文件的麻烦。

---

## 超时 + 指数退避重试模板

任何时候调用外部命令（网络请求、远程 ssh、锁获取），都该有超时和重试。

```bash
retry_with_backoff() {
    local max_attempts=${1:-5}
    local cmd="${*:2}"
    local attempt=0
    local delay=1

    until [[ $attempt -ge $max_attempts ]]; do
        if eval "$cmd"; then
            return 0
        fi
        ((attempt++))
        echo "[WARN] Attempt $attempt/$max_attempts failed, retrying in ${delay}s..." >&2
        sleep "$delay"
        delay=$(( delay * 2 ))          # 1 → 2 → 4 → 8 → 16
    done

    echo "[ERROR] All $max_attempts attempts failed" >&2
    return 1
}

# 用法
retry_with_backoff 3 'curl -sf https://api.example.com/health'
retry_with_backoff 5 'ssh db01 "pg_isready"'
```

**要点**：`eval` 用在已知可控的命令上没问题，如果命令来自外部输入则需改用数组传参。

---

## 沉默是金：成功不输出，失败才写 stderr

这是 Unix 最经典的设计哲学，但也最容易被打破。

```bash
# 不要这样
deploy_app() {
    echo "Starting deployment..."
    rsync -avz ./dist/ app-server:/opt/app/
    echo "Deployment done!"
    # 调用者如果想在脚本里用这个函数，没法区分"正常输出了 done"和"真的成功了"
}

# 应该这样
deploy_app() {
    rsync -avz ./dist/ app-server:/opt/app/  # rsync 自己会打印同步内容
    # 成功时不额外输出，调用者自然知道
    # 失败时 rsync 自己写 stderr，不需要你操心
}
```

什么时候该输出：
- **严重错误** → 写到 stderr，附带 `[ERROR]` 前缀和时间戳
- **关键状态变更** → 写到 stderr（因为管道的 stdout 可能被重定向到文件/被下游消费）
- **默认路径** → 什么都不写。谁调用谁决定是否加 `-v`

```bash
# 如果需要进度反馈，加到 stderr，不污染 stdout
convert_image() {
    local input="$1" output="$2"
    convert "$input" -resize 800x600 "$output"
    echo "[OK] $input → $output" >&2   # 写到 stderr
}
```
