# 7. 排障

> 脚本总会挂的。问题不是"会不会挂"，而是"挂了之后我能不能快速知道为什么"。

---

## 出错时自动抓现场快照

与其等人拿着报错截图来找你，不如让脚本在挂掉的那一刻自己把现场留下。

```bash
SNAPSHOT_DIR="/var/log/myapp/diagnostics"

snapshot() {
    local ts
    ts=$(date +'%Y%m%d-%H%M%S')
    local dir="$SNAPSHOT_DIR/$ts"

    mkdir -p "$dir"
    {
        echo "=== Snapshot at $ts ==="
        echo "Script: $0"
        echo "Line: $BASH_LINENO"
        echo "Exit code: $?"
        echo "---"
        ps aux | grep -E "(myapp|$$)" > "$dir/process.txt"
        df -h > "$dir/disk.txt"
        free -m > "$dir/memory.txt"
        dmesg | tail -20 > "$dir/dmesg.txt"
        tail -100 "$LOGFILE" > "$dir/tail.log" 2>/dev/null || true
    } > "$dir/report.txt" 2>&1

    echo "[FATAL] Snapshot saved to $dir" >&2
}

trap 'snapshot' ERR
```

**要点**：
- 时间戳目录，不覆盖之前的快照
- 带进程状态、磁盘、内存、日志尾巴
- 只抓和当前进程相关的信息，不 dump 全系统

---

## 调试开关：PS4 + BASH_LINENO + BASH_SOURCE

`set -x` 默认的输出几乎不可读：

```
+ echo hello
+ some_func
++ date
```

**调教过的版本**：

```bash
export PS4='+ [${BASH_SOURCE}:${LINENO}] ${FUNCNAME[0]:+${FUNCNAME[0]}():} '
set -x

# 输出变成：
# + [script.sh:12] main(): echo hello
# + [script.sh:5] some_func(): date
```

哪个文件、哪一行、哪个函数，一目了然。

**按需开关**：

```bash
DEBUG=${DEBUG:-0}

if [[ "$DEBUG" == "1" ]]; then
    export PS4='+ [${BASH_SOURCE##*/}:${LINENO}] '
    set -x
fi
```

---

## 脚本崩溃时让人类知道该看哪一行

`trap ERR + LINENO` 是零日志错误定位的利器：

```bash
trap 'echo "[FATAL] line $LINENO: unexpected exit ($?)" >&2; exit 1' ERR
set -Eeuo pipefail

# 如果任意命令失败，精确告诉你哪一行
# 例如 output:
# [FATAL] line 42: unexpected exit (1)
```

**结合函数调用栈**（更详细的上下文）：

```bash
fatal() {
    local rc=$?
    local line=$1
    echo "[FATAL] line $line: exit code $rc" >&2
    local i
    for ((i = 1; i < ${#FUNCNAME[@]}; i++)); do
        echo "  called from ${BASH_SOURCE[$i]}:${BASH_LINENO[$i-1]} ${FUNCNAME[$i]}()" >&2
    done
    exit "$rc"
}

trap 'fatal $LINENO' ERR
set -Eeuo pipefail
```

输出示例：

```
[FATAL] line 42: exit code 1
  called from deploy.sh:18 deploy_app()
  called from deploy.sh:55 main()
```

有了这个，不需要在每行后面写 `|| log_error`。少写是福。
