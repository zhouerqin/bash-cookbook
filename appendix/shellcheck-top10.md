# 附录：ShellCheck Top 10 最疼规则

> ShellCheck 是个好工具，但 300+ 条规则记不住。以下是在生产代码里最容易踩的 10 条。

---

| # | 规则 | 代码 | 问题 | 修复 |
|---|------|------|------|------|
| 1 | SC2086 | `echo $var` | 变量没引用，空格/通配符炸裂 | `echo "$var"` |
| 2 | SC2206 | `arr=($(cmd))` | 未引用的命令替换 + 数组赋值，空格分裂 | `readarray -t arr < <(cmd)` 或 `IFS= read -ra` |
| 3 | SC2046 | `rm $(find ...)` | find 输出分词，文件名带空格 rm 错对象 | `find ... -print0 \| xargs -0 rm` |
| 4 | SC2162 | `read line` | `read` 默认会 trim IFS 字符 | `IFS= read -r line` |
| 5 | SC2155 | `local x=$(cmd)` | `local` 的返回值被 `declare` 覆盖 | `local x; x=$(cmd)` |
| 6 | SC2002 | `cat file \| grep` | 无用的 cat | `grep pattern file` |
| 7 | SC2005 | `echo $(cmd)` | 多余的命令替换 | 直接 `cmd` |
| 8 | SC2016 | `echo '$var'` | 单引号会阻止变量展开——如果你真想展开的话 | 确认意图，双引号或单引号 |
| 9 | SC2128 | `echo $arr` | 数组变量不加 `[@]` 只取第一个元素 | `echo "${arr[@]}"` |
| 10 | SC2251 | `set +e` 后没恢复 | 关闭 `set -e` 后忘记恢复，后续错误全部静默 | 用子 shell `(set +e; cmd)` 或记着 `set -e` |

---

## 正确使用示例

```bash
# SC2155 正确写法
my_func() {
    local result     # 先声明
    result=$(some_command)  # 再赋值
    echo "$result"
}

# SC2046 / SC2086 正确写法
find /tmp -name "*.tmp" -mtime +7 -print0 | xargs -0 rm -f
```

建议在 CI 中跑 `shellcheck -s bash -f gcc yourscript.sh`，白名单模式下只关掉你确认无影响的规则。
