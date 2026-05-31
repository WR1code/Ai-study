1. 安装基础工具

   
sudo apt update

sudo apt install -y curl git ca-certificates

3. 安装 Codex CLI

官方推荐的 Linux/macOS 安装命令是：

curl -fsSL https://chatgpt.com/codex/install.sh | sh

官方也提供非交互安装方式：

curl -fsSL https://chatgpt.com/codex/install.sh | CODEX_NON_INTERACTIVE=1 sh

安装完成后检查：

codex --version

如果提示 codex: command not found，先重新加载 shell：

source ~/.bashrc

或者重新打开一个终端。

Codex 也可以用 npm 安装：

npm install -g @openai/codex

不过在 Ubuntu 上我更建议优先用官方 standalone 安装脚本，少折腾 Node/npm 环境。官方 GitHub 也列出了这两种安装方式。

3. 进入你的工程目录

比如你的 ROS2 小车项目：

cd /root/intelligent_car_ws

然后启动 Codex：

codex

第一次会让你登录。官方说明支持 ChatGPT 账号 或 API key 登录；Plus、Pro、Business、Edu、Enterprise 计划包含 Codex 访问。

4. 最安全的使用方式

先让它只在当前项目里工作：

cd /root/intelligent_car_ws

codex --sandbox workspace-write --ask-for-approval on-request

这个模式的意思是：Codex 可以在当前工作区读写文件，但遇到一些敏感命令会向你确认。官方说明里也把 workspace-write + on-request 作为显式安全配置示例。

进去以后你可以直接输入中文任务，例如：

帮我检查这个 ROS2 工作空间，找出为什么 ros2 launch origincar_avoid all_detect.launch.py 报错，并给出修改方案

或者：

帮我查看 origincar_avoid 包，把 launch、CMakeLists.txt、package.xml 的安装路径检查一遍

5. 让它“自动工作”的命令模式

如果你想让它不打开交互界面，直接执行一个任务，用：

codex exec "检查这个 ROS2 工作空间的编译错误，并给出修复建议"

官方文档说 codex exec 是非交互模式，适合脚本、CI、流水线任务，不打开 TUI。

如果你想让它能自动改文件，但仍限制在当前工程：

codex exec --sandbox workspace-write "修复 origincar_avoid 包中的 launch 导入错误，并说明你改了哪些文件"

不要一上来用这个：

codex exec --dangerously-bypass-approvals-and-sandbox "..."

官方文档明确说这个会绕过审批和沙箱，只建议在隔离 runner 里使用。




. 检查 Ubuntu 的 AppArmor 限制

你这台机器看起来是 Ubuntu 桌面系统，很可能是 Ubuntu 24.04 / 25.04 之后的限制。

先看这两个值：

cat /proc/sys/kernel/apparmor_restrict_unprivileged_userns
cat /proc/sys/kernel/apparmor_restrict_unprivileged_unconfined

如果输出是：

1

1

就说明 AppArmor 对非特权 user namespace 有限制。Codex 官方文档说，Ubuntu 24.04 可能需要加载额外的 bwrap AppArmor profile；Ubuntu 25.04 中相关 profile 通常随 apparmor 包提供。


2. 检查 user namespace 是否开启

执行：

cat /proc/sys/kernel/unprivileged_userns_clone

如果输出是：

1

说明这一项没问题。







你的输出说明：

```text
unprivileged_userns_clone = 1
apparmor_restrict_unprivileged_userns = 1
apparmor_restrict_unprivileged_unconfined = 0
```

所以问题不是 `user namespace` 没开，而是 **AppArmor 限制了 bubblewrap 创建非特权 user namespace**。Codex 官方说明 Ubuntu 24.04 上即使安装了 `bubblewrap`，仍可能提示这个警告，需要加载 `bwrap-userns-restrict` AppArmor profile。([OpenAI开发者][1])

你按这个来处理。

## 方案一：推荐，加载 bwrap 的 AppArmor 配置

先退出 Codex，然后执行：

```bash
sudo apt update
sudo apt install -y apparmor-profiles apparmor-utils
```

然后看这个文件在不在：

```bash
ls /usr/share/apparmor/extra-profiles/bwrap-userns-restrict
```

如果能看到文件，就执行：

```bash
sudo install -m 0644 \
  /usr/share/apparmor/extra-profiles/bwrap-userns-restrict \
  /etc/apparmor.d/bwrap-userns-restrict

sudo apparmor_parser -r /etc/apparmor.d/bwrap-userns-restrict
```

再重新进 Codex：

```bash
cd ~/ros2_ws
codex
```

官方说明 `apparmor_parser -r` 会把 profile 加载进内核，不需要重启。([OpenAI开发者][1])

---

## 方案二：如果方案一找不到文件

执行：

```bash
ls /etc/apparmor.d/ | grep bwrap
```

如果已经有：

```text
bwrap-userns-restrict
```

那就直接加载：

```bash
sudo apparmor_parser -r /etc/apparmor.d/bwrap-userns-restrict
sudo systemctl reload apparmor.service
```

然后重新打开：

```bash
codex
```

---

## 方案三：临时关闭这个限制，不太推荐但最直接

如果前两个方案都不行，可以临时关闭 AppArmor 对非特权 user namespace 的限制：

```bash
sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
```

然后：

```bash
cd ~/ros2_ws
codex
```

这个方法官方也给了，但它是全局放开限制，安全性比加载专门的 `bwrap` profile 差一点。([OpenAI开发者][1])

---

你现在优先执行这几条就行：

```bash
sudo apt update
sudo apt install -y apparmor-profiles apparmor-utils

sudo install -m 0644 \
  /usr/share/apparmor/extra-profiles/bwrap-userns-restrict \
  /etc/apparmor.d/bwrap-userns-restrict

sudo apparmor_parser -r /etc/apparmor.d/bwrap-userns-restrict

cd ~/ros2_ws
codex
```

如果 `install` 那一步提示 **No such file or directory**，把报错发我。

[1]: https://developers.openai.com/codex/concepts/sandboxing?utm_source=chatgpt.com "Sandbox – Codex"

