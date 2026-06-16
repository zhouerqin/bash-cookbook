# 5. 进程管理

> 写运维脚本本质就是管进程的生死。

---

## pidfile 的竞态与真正可靠的存活检测

新手写服务管理脚本的第一反应是 pidfile 模式。但它有经典竞态：

```bash
# 不要这样写
echo $$ > /var/run/myapp.pid
# 问题：两个脚本同时执行，同时写同一个文件，各自以为自己独占
# 原子性无法保证
```

**靠谱方案一：`flock`**（推荐）

```bash
exec 200>/var/run/myapp.lock
flock -n 200 || { echo "Already running" >&2; exit 1; }

# 此时你的进程是唯一的
# 无需 pidfile，flock 基于内核文件锁，进程退出自动释放

# 如果需要知道 PID，可以写入锁文件内部
echo $$ >&200
```

**靠谱方案二：`pidfile` + `kill -0` 配合检查**（必须要用 pidfile 时）

```bash
write_pidfile() {
    local pidfile="$1"
    local old_pid

    # 先读取旧的 pid
    if [[ -f "$pidfile" ]]; then
        old_pid=$(<"$pidfile")
        # 检查旧的进程是否存在且确实是我们的进程
        if [[ -n "$old_pid" ]] && kill -0 "$old_pid" 2>/dev/null; then
            # 进一步确认进程名（防止 pid 被复用）
            if grep -z "myapp" "/proc/$old_pid/cmdline" &>/dev/null; then
                echo "Process already running (PID: $old_pid)" >&2
                exit 1
            fi
        fi
    fi

    # 原子写入（先写临时文件再 mv）
    echo $$ > "${pidfile}.$$"
    mv "${pidfile}.$$" "$pidfile"
}
```

**存活检测的真正可靠方式**：不用 pidfile，而是用 `systemd` 或 `supervisord` 这类进程管理器来管。它们比 shell 脚本靠谱得多。你的脚本只需要负责**告诉管理器启动/停止**。

---

## 服务启停的幂等设计

对生产环境运行的操作必须支持反复执行不产生副作用。

```bash
# 幂等的启动
start_service() {
    local svc="$1"
    if systemctl is-active --quiet "$svc"; then
        echo "[OK] $svc already running" >&2
        return 0
    fi
    systemctl start "$svc"
    echo "[OK] $svc started" >&2
}

# 幂等的停止
stop_service() {
    local svc="$1"
    if ! systemctl is-active --quiet "$svc"; then
        echo "[OK] $svc already stopped" >&2
        return 0
    fi
    systemctl stop "$svc"
    echo "[OK] $svc stopped" >&2
}

# 幂等的重启（不管当前状态，保证运行）
ensure_running() {
    local svc="$1"
    if ! systemctl is-active --quiet "$svc"; then
        systemctl start "$svc"
    fi
    # 等待确认
    local wait=0
    until systemctl is-active --quiet "$svc"; do
        sleep 1
        ((wait++))
        [[ $wait -le 10 ]] || { echo "[ERROR] $svc failed to start" >&2; return 1; }
    done
}
```

**非 systemd 环境下的幂等启动**：

```bash
start_daemon() {
    local name="$1" cmd="$2" pidfile="$3"

    if [[ -f "$pidfile" ]]; then
        local pid
        pid=$(<"$pidfile")
        if kill -0 "$pid" 2>/dev/null; then
            echo "[OK] $name already running (PID: $pid)" >&2
            return 0
        fi
        echo "[WARN] Stale pidfile found, removing" >&2
        rm -f "$pidfile"
    fi

    nohup "$cmd" >/dev/null 2>&1 &
    echo $! > "$pidfile"
    echo "[OK] $name started (PID: $!)" >&2
}
```
