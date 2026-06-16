# 附录：常备切片

> 写到一半发现又要写一段重复的代码。以下是你每次写新脚本都可以直接复制粘贴的片段。

---

## 严格模式 + 错误追踪（防呆版）

```bash
#!/bin/bash
set -Eeuo pipefail

trap 'echo "[FATAL] line $LINENO: exit code $?" >&2; exit 1' ERR
```

建议所有新脚本直接从这段开始写。

---

## 脚本路径解析（防软链接）

```bash
SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd -P)"
SCRIPT_NAME="$(basename -- "${BASH_SOURCE[0]}")"
```

---

## 临时目录（自动清理）

```bash
TMPDIR=$(mktemp -d) || { echo "mktemp failed" >&2; exit 1; }
trap 'rm -rf -- "$TMPDIR"' EXIT
```

---

## 单实例锁

```bash
LOCKFILE="/var/run/${SCRIPT_NAME}.lock"
exec 200>"$LOCKFILE"
flock -n 200 || { echo "Another instance is running" >&2; exit 1; }
```

---

## 超时重试（指数退避）

```bash
retry() {
    local max=${1:-5} cmd="${*:2}" n=0 d=1
    until [[ $n -ge $max ]]; do
        eval "$cmd" && return 0
        ((n++)); sleep "$d"; d=$((d * 2))
    done
    echo "[ERROR] All $max attempts failed" >&2
    return 1
}
```

---

## 并行执行（批量等待）

```bash
PIDS=()
for host in "${HOSTS[@]}"; do
    (ssh "$host" "$@") & PIDS+=($!)
done
FAILED=0
for pid in "${PIDS[@]}"; do
    wait "$pid" || ((FAILED++))
done
exit "$FAILED"
```

---

## 日志脱敏

```bash
sanitize() {
    sed -E \
        -e 's/(password|passwd|secret|token|api_key)(["=: ]+)[^"& ]+/\1\2***/gi' \
        -e 's/(Authorization: Basic )[^ ]+/\1***/gi' \
        -e 's/(Bearer )[a-zA-Z0-9._-]+/\1***/gi'
}
```

---

## 调试输出（带行号和文件名）

```bash
DEBUG=${DEBUG:-0}
if [[ "$DEBUG" == "1" ]]; then
    export PS4='+ [${BASH_SOURCE##*/}:${LINENO}] '
    set -x
fi
```
