# Bash 手册 / bash-cookbook

> 写给 18 年前的自己。不解释 `if`，只说那些书上看不到的。
>
> 两条信条：管道组合 OS 进程 | 简洁本身就是可维护性
>
> Shell 最强大的能力不是语法，而是把 OS 进程当一等公民做函数式组合。

## 目录

[1. 组合（Composition）](01-composition.md)
- 管道不是语法糖——是 OS 级别的函数式编程
- `<( )` 进程替换：把远程主机/命令当文件用
- `|&` 同时转发 stdout + stderr
- 管道的 SIGPIPE：区分"被掐断"和"真报错"
- `&` + `wait` + `$!`：并行编排通用模板
- 管道与 `set -o pipefail` 的微妙关系

[2. 防御性编程](02-defensive.md)
- 单实例锁（flock）的正确写法
- 管道被吞的错误如何不漏
- 临时文件的安全创建与退出清理
- 超时 + 指数退避重试模板
- 沉默是金：成功不输出，失败才写 stderr

[3. 信号处理](03-signals.md)
- 为什么 `trap cleanup EXIT` 还不够
- 容器内 `exec` 的信号转发
- 子进程组的兜底清理

[4. 字符串与路径](04-strings-paths.md)
- 路径操作的三个深坑（软链接、空格、未定义展开）
- 用 `envsubst` 而不是拼 `sed`
- `find -print0` + `read -d ''` 的空安全流水线

[5. 进程管理](05-process.md)
- pidfile 的竞态与真正可靠的存活检测
- 服务启停的幂等设计

[6. 安全](06-security.md)
- 密码不从命令行参数进入
- `mktemp` 的 umask 陷阱
- 日志脱敏通用函数

[7. 排障](07-troubleshoot.md)
- 出错时自动抓现场快照
- 调试开关（PS4 + BASH_LINENO + BASH_SOURCE）
- `trap ERR + LINENO`：不用打日志就知道错在哪一行

[8. 简洁之道](08-simplicity.md)
- 沉默是金
- `trap ERR + LINENO`：零日志的精确错误定位
- 最小的可维护单元
- 函数设计的单一职责
- 信任上游，就地失败

## 附录

- [常备切片（可复用的代码段）](appendix/snippets.md)
- [ShellCheck Top 10 最疼规则](appendix/shellcheck-top10.md)
