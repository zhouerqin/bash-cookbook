# 6. 安全

> Shell 脚本最容易出安全问题的不是注入，而是"顺手"。

---

## 密码不从命令行参数进入

`ps aux` 能看到所有进程的命令行参数。把密码放参数里，同机所有用户都能看到。

```bash
# 错误——所有人能看到密码
mysql -u root -pMySecretPass123 -e "SHOW DATABASES"

# 也是错误——环境变量在 /proc/PID/environ 一样可见
export DB_PASS="MySecretPass123"
mysql -u root -p"$DB_PASS" -e "SHOW DATABASES"
```

**正确做法**：从文件读，用完即焚。

```bash
# 方案一：配置文件（600 权限）
DB_PASS=$(cat /etc/db-pass.txt)
mysql -u root -p"$DB_PASS" -e "SHOW DATABASES"

# 方案二：交互式输入
read -r -s -p "Password: " DB_PASS
mysql -u root -p"$DB_PASS" -e "SHOW DATABASES"

# 方案三：默认凭证文件
mysql --defaults-extra-file=/etc/mysql/my.cnf -e "SHOW DATABASES"
# my.cnf 内容：
# [client]
# password=MySecretPass123
```

---

## `mktemp` 的 umask 陷阱

`mktemp` 创建的临时文件默认权限是 **600**（只有创建者可读写）——这符合安全预期。

但是如果你设了 umask：

```bash
umask 0022
TMPFILE=$(mktemp)
ls -l "$TMPFILE"
# -rw------- 1 root root ...   ← 仍然是 600，mktemp 不受 umask 影响
```

然而有一个例外：**`mktemp -d` 创建的目录**，当你在里面创建文件时，这些文件的权限受 umask 影响：

```bash
umask 0002   # 组可写
TMPDIR=$(mktemp -d)
touch "$TMPDIR/secret"
ls -l "$TMPDIR/secret"
# -rw-rw-r--  ...  ← 组内其他用户可以读！
```

**安全做法**：创建临时目录后显式收紧权限：

```bash
TMPDIR=$(mktemp -d) || exit 1
chmod 700 "$TMPDIR"
trap 'rm -rf "$TMPDIR"' EXIT
```

---

## 日志脱敏通用函数

脚本里不小心把密码/Token 打印到日志里，是生产环境最常见的信息泄露。

```bash
# 脱敏函数
sanitize() {
    # 替换常见的敏感字段值
    sed -E \
        -e 's/(password|passwd|secret|token|api_key)(["=: ]+)[^"& ]+/\1\2***/gi' \
        -e 's/(Authorization: Basic )[^ ]+/\1***/gi' \
        -e 's/(Bearer )[a-zA-Z0-9._-]+/\1***/gi'
}

# 用法
log_info() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] INFO: $*" | sanitize >&2
}

# 也可以在关键命令后脱敏
curl -sv -H "Authorization: Bearer $TOKEN" https://api.example.com 2>&1 | sanitize >> "$LOGFILE"
```
