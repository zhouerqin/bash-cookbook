# 3. 信号处理

> 写 Shell 信号处理更像是写一个"不留后患"的契约，而不是"优雅停机"——进程没那么优雅。

---

## 为什么 `trap cleanup EXIT` 还不够

`trap 'cleanup' EXIT` 是最常见的写法，它能覆盖正常退出、`exit`、`SIGINT`、`SIGTERM`。

但有一个场景它会失效：**`set -e`**。

```bash
set -e
trap 'echo "CLEANUP"' EXIT

some_command() {
    false   # 返回非零
    echo "never reached"
}

some_command
# set -e 会让整个脚本退出，trap EXIT 会触发，看起来没问题
```

再看这个：

```bash
set -e
trap 'echo "CLEANUP"' EXIT

# 管道中的失败
false | true
# pipefail 没开，$?=0，没问题

# 但如果是这样
set -o pipefail
false | true
# $?=1，但是 trap EXIT 依然会触发
```

**真正的问题**在**子 shell 不继承 trap**：

```bash
trap 'echo "CLEANUP"' EXIT

# 子 shell 里没有 EXIT trap
(
    mktemp -d /tmp/sub-XXX
    # 这个子 shell 如果挂掉，它创建的临时文件不会被清理
)
```

**解法**：在函数内部也设 trap：

```bash
safe_subshell() {
    local tmpdir
    tmpdir=$(mktemp -d)
    trap 'rm -rf "$tmpdir"' RETURN   # RETURN 在函数返回时触发，包括错误返回
    # ... 干活 ...
}

# 或者在子 shell 内显式 trap
(
    tmpdir=$(mktemp -d)
    trap 'rm -rf "$tmpdir"' EXIT
    # ... 干活 ...
)
```

---

## 容器内 `exec` 的信号转发

Docker 容器里 PID 1 如果是一个 shell 脚本，它**不转发信号给子进程**。这是最容易被忽视的容器问题。

```bash
# 错误写法
#!/bin/bash
some_long_running_process
# docker stop 发 SIGTERM 给这个 shell，shell 不转发给 some_long_running_process
# 10 秒后 docker 发 SIGKILL，进程被强杀
```

**正确写法**：

```bash
#!/bin/bash
# 用 exec 替换当前进程，让子进程成为 PID 1
exec some_long_running_process
# 现在 docker stop 的 SIGTERM 直接发给 some_long_running_process
# 它能优雅处理自己的信号
```

如果需要启动多个进程：

```bash
#!/bin/bash
# 用 exec 启动一个 init 风格的包装
exec dumb-init -- /entrypoint.sh
# 或者用 tini，或者手动 trap + wait

trap 'kill -TERM $PID; wait' SIGTERM SIGINT
some_long_running_process &
PID=$!
wait
```

---

## 子进程组的兜底清理

启动后台进程后脚本退出，后台进程变成孤儿，被 init 收养。生产环境上这是**日志文件写爆炸**的常见原因。

```bash
# 错误写法
start_worker() {
    nohup python worker.py &
    echo "Worker started"
    exit 0
    # shell 退出了，worker.py 变成孤儿继续跑
}
```

**正确做法**：

```bash
start_worker() {
    # 把子进程加入同一个进程组
    set -m            # 启用 monitor mode（作业控制）
    python worker.py &
    WPID=$!
    echo "$WPID" > /var/run/worker.pid

    # 在退出脚本时清理
    trap 'kill -TERM -$WPID 2>/dev/null; wait' EXIT
    # 负号表示发给整个进程组
}
```

**通用兜底模式**：启动时在临时目录记下所有后台 PIDs，清理时逐个确认。

```bash
PIDS_FILE="$TMPDIR/background_pids"

launch_bg() {
    "$@" &
    echo "$!" >> "$PIDS_FILE"
}

cleanup_bg() {
    [[ -f "$PIDS_FILE" ]] || return 0
    while IFS= read -r pid; do
        kill -0 "$pid" 2>/dev/null && kill -TERM "$pid" 2>/dev/null || true
    done < "$PIDS_FILE"
}

trap 'cleanup_bg' EXIT

# 用法
launch_bg python worker1.py
launch_bg python worker2.py
```
