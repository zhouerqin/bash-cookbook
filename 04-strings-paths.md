# 4. 字符串与路径

> 大部分 Shell 脚本的运行时错误不是逻辑问题，是路径没处理好。

---

## 路径操作的三个深坑

### 坑一：软链接

`$(dirname "$0")` 拿到的是脚本**被调用的路径**，不是脚本**真正的位置**。

```bash
# /usr/local/bin/myscript → /opt/app/bin/myscript.sh
# 执行 myscript，$0 是 /usr/local/bin/myscript
# dirname 得到 /usr/local/bin，不是 /opt/app/bin
```

**解法**：

```bash
SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd -P)"
# -P 选项：解析所有软链接，拿到真实路径
```

### 坑二：空格

```bash
DIR="/path/My Projects"
cd $DIR       # 错误：cd 收到两个参数
cd "$DIR"     # 正确：作为一个参数传入
```

空格在 for 循环里尤其隐蔽：

```bash
# 错误
for f in $(find /path -type f); do
    stat "$f"
done
# 文件名带空格 → 被拆成多个参数

# 正确
find /path -type f -print0 | while IFS= read -r -d '' f; do
    stat "$f"
done
```

### 坑三：未定义变量展开

```bash
# 变量没定义，也不会报错
rm -rf /opt/$APPNAME/logs/
# 如果 $APPNAME 未定义 → rm -rf /opt/logs/ 还在可控范围
# 如果 $APPNAME 为空   → rm -rf /opt/logs/ 同上
# 但如果是 rm -rf /opt/$APPNAME 且 $APPNAME 为空 → rm -rf /opt/ ！
```

**解法**：`set -u` 让未定义变量直接报错退出，或者显式给默认值：

```bash
: "${APPNAME:?APPNAME is not set}"
# 或
rm -rf "/opt/${APPNAME:?}/logs/"
```

---

## 用 `envsubst` 而不是拼 `sed`

需要把配置文件中的占位符替换为实际值时，不要手写 `sed`。

```bash
# 不要这样
sed "s/{{HOST}}/$HOST/g; s/{{PORT}}/$PORT/g; s/{{DB_URL}}/$DB_URL/g" config.tpl > config.cfg

# 应该这样
export HOST PORT DB_URL
envsubst < config.tpl > config.cfg
```

**原因**：
- `sed` 容易踩分隔符冲突的坑（`$HOST` 里的 `/` 会让 `sed` 崩溃）
- `envsubst` 只替换 `$VAR` 和 `${VAR}` 格式，不会动其他内容
- 模板看起来一目了然：`host=$HOST port=$PORT`

**如果需要限定只替换某些变量**（避免把环境变量全部炸开）：

```bash
# 只替换 $HOST 和 $PORT
envsubst '$HOST $PORT' < config.tpl > config.cfg
```

---

## `find -print0` + `read -d ''` 的空安全流水线

这是处理文件名最安全的方案，没有之一。

```bash
# 不安全
for f in *.log; do ...           # 文件名带空格崩

# 也不安全
find . -name "*.log" | while read f; do ...  # 换行符 / 空格崩

# 安全
find . -name "*.log" -print0 | while IFS= read -r -d '' f; do
    echo "Processing: $f"
done
```

**原理**：
- `-print0`：用 NUL（`\0`）分隔文件名——这是唯一不会出现在文件名中的字符
- `read -d ''`：告诉 read 以 NUL 为分隔符
- `IFS=`：防止 read 修剪首尾空白

**批量操作时的安全写法**：

```bash
# 删除 30 天前的日志
find /var/log/myapp -name "*.log" -mtime +30 -print0 | xargs -0 rm -f
# -0 告诉 xargs 以 NUL 分隔

# 复制到备份目录
find /data -name "*.db" -print0 | while IFS= read -r -d '' f; do
    cp "$f" /backup/"$(basename "$f")"
done
```
