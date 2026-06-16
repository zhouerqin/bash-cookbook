# 8. 简洁之道

> 好的 Shell 脚本读起来像在说话。不好的 Shell 脚本读起来像在排雷。

---

## 沉默是金

成功时不输出。失败时写 stderr。这是 Unix 的哲学，也是 Shell 脚本可组合性的前提。

```bash
# 坏的：成功路径的打印让管道没法用
check_port() {
    if nc -z "$1" "$2"; then
        echo "Port $2 is open"
        return 0
    else
        echo "Port $2 is closed" >&2
        return 1
    fi
}

# 好的：成功时不输出，调用者自己决定是否加 -v
check_port() {
    nc -z "$1" "$2"
}
```

**判断标准**：在终端跑你的函数之后，能不能自然地接到管道里？如果能，你的输出做对了。

---

## `trap ERR + LINENO`：零日志的精确错误定位

不需要在每个可能出错的命令后面加 `|| echo "failed"`。

```bash
# 坏：每行都要手工写日志，还容易漏
mkdir -p /tmp/workdir || { echo "mkdir failed"; exit 1; }
cd /tmp/workdir || { echo "cd failed"; exit 1; }
curl -sf http://example.com || { echo "curl failed"; exit 1; }

# 好：一次设置，覆盖全文
trap 'echo "[FATAL] line $LINENO: exit code $?" >&2; exit 1' ERR
set -Eeuo pipefail
mkdir -p /tmp/workdir
cd /tmp/workdir
curl -sf http://example.com
```

**为什么简洁**：错误处理集中在一行，代码主体全是"happy path"，读起来像是对照流程文档。

---

## 最小的可维护单元：能一行搞定的不写成函数

```bash
# 不需要包装
get_timestamp() {
    date +'%Y-%m-%d %H:%M:%S'
}

# 直接写
ts=$(date +'%Y-%m-%d %H:%M:%S')

# 或者只在需要的时候再绑别名/变量
TIMESTAMP_FMT='%Y-%m-%d %H:%M:%S'
```

**判断标准**：如果一个函数体比它的函数名 + 参数声明还短，它就不该是函数。

---

## 函数设计的单一职责

一个函数只做一件事。名字就是注释。

```bash
# 坏：一个函数干了三件事
deploy() {
    git pull
    npm install
    npm run build
    systemctl restart myapp
    # 如果 npm install 失败，调用者不知道是哪一步出的问题
}

# 好：职责拆开，可单独调用和测试
git_pull()      { git pull; }
install_deps()  { npm install; }
build_app()     { npm run build; }
restart_app()   { systemctl restart myapp; }

deploy() {
    git_pull && install_deps && build_app && restart_app
}
```

**收益**：
- 失败时 `trap ERR` 直接报第几行，立刻知道哪个阶段
- 每个步骤可以独立测试：`install_deps && build_app`
- 可以只跑某个步骤：`restart_app` 不重新部署

---

## 避免防御过度——信任上游，就地失败

新手怕出错，所以在每个命令前加检查。老手知道：**如果上游该保证的东西没保证，那是上游的问题，你的脚本不需要再检查一遍**。

```bash
# 坏的：过度防御
if [[ -f /etc/myapp/config.yml ]]; then
    content=$(cat /etc/myapp/config.yml)
    if [[ -n "$content" ]]; then
        process_config <<< "$content"
    else
        echo "Config file is empty" >&2
        exit 1
    fi
else
    echo "Config file not found" >&2
    exit 1
fi

# 好的：信任配置管理工具已经保证了文件存在，不存在就自然失败
set -Eeuo pipefail
process_config < /etc/myapp/config.yml
# cat 都不需要，< 直接读
```

如果文件不存在，`/etc/myapp/config.yml` 打开失败 → shell 报错 → `ERR` trap 触发 → 精确报行号。**够了**。

**信任链条**：
- 配置管理（Ansible/Puppet）保证文件存在 → 脚本不重复检查
- 上游管道保证输入格式 → 下游不重复校验
- DNS 保证主机名可解析 → ssh 不预检联通性

只在和外部世界交互的边界做检查（用户输入、网络请求、磁盘空间），内部路径信任自然失败即可。
