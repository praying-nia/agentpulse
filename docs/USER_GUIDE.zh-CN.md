# AgentPulse 使用手册

适用范围：当前源码中的 AgentPulse Host、Android App，以及可选的自建 Relay。最后核对：2026-09-10。

AgentPulse 让你在 Android 手机上查看电脑端 Codex 的任务进展、处理审批和问题、发送消息与常用控制指令。电脑负责运行 Codex，手机负责连接和交互；关闭手机连接不会自动停止电脑上的任务。

本文从首次安装讲起。已有环境的用户可直接阅读“日常使用”“手机操作”或“故障排查”。所有 `<...>` 都是需要替换的参数；`host.example.com`、`relay.example.com` 和 `203.0.113.10` 都是示例，不能原样用于连接。

## 目录

1. [支持范围与准备工作](#requirements)
2. [选择连接方式](#routes)
3. [安装电脑端与 Android App](#installation)
4. [首次使用：已有 Relay](#relay-quickstart)
5. [首次使用：公网直连](#direct-quickstart)
6. [扫码配对的完整过程](#pairing)
7. [日常启动、退出与 Linux 快捷命令](#daily)
8. [手机上的会话、审批和消息](#phone)
9. [手机指令速查](#commands)
10. [连接方式、配置修改与重连](#reconnect)
11. [设备管理、数据保存与升级](#maintenance)
12. [自建 Relay](#self-hosted-relay)
13. [故障排查](#troubleshooting)
14. [电脑端命令速查与验收步骤](#reference)

<a id="requirements"></a>
## 1. 支持范围与准备工作

### 当前可以使用什么

| 项目 | 当前范围 |
| --- | --- |
| AI 工具 | Codex CLI，通过 AgentPulse 管理的 App Server 接入 |
| 电脑端 | Linux、Windows、macOS 的 Host 实现；不同平台验证进展见开发日志 |
| 手机端 | Android 8.0 / API 26 及以上，需要摄像头完成首次配对 |
| 手机交互 | 会话与事件、计划、审批、结构化回答、普通消息、本文列出的控制指令 |
| 公网连接 | Relay 转发，或显式配置的 Host 公网直连 |
| iOS、HarmonyOS、其他 AI 工具与 Bot 通道 | 尚不是本文可以照做的完整产品入口 |

已在本机网卡直挂公网 IPv4、Android 15 手机移动网络下完成二维码配对、业务连接、App 冷启动及重连验收；保持受管控制连接在线时，真实消息往返也已通过。实际端口映射、IPv6 与摄像头光学扫码尚未覆盖。控制连接退出后的消息回传另有待排查情况，见[实机验收记录](validation/2026-09-10-public-direct.md)。你的防火墙和公网入口仍需自行验证。

### 电脑端准备

- 安装并配置好可独立使用的 Codex CLI。AgentPulse 不替你创建 Codex 账户或设置其认证。
- 能在终端运行 `codex --version`，并能通过该 Codex 环境正常工作。
- 使用能持续运行任务的电脑或服务器；机器休眠、关机或 Host 停止后，手机无法继续访问它。
- 准备一种连接方式：Relay 地址与登记令牌，或者 Host 对手机可达的公网入口。

本仓库明确列出的 Codex 版本为 `0.150.1`、`0.152.0`、`0.152.1`、`0.153.0`。更高的合法版本号会在提示后尽力运行，但仍可能因为接口变化而失败；这不是对所有新版本的兼容承诺。具体约束见 [Codex Provider 说明](../agentpulse-rs/agentpulse-providers/codex/README.md)。

### 手机端准备

安装 AgentPulse，按需允许相机和通知权限。选择局域网路线时，系统若请求本地网络权限，也需要允许。手机通过互联网扫码配对，不需要 USB、ADB 或蓝牙；ADB 只是开发测试工具。

<a id="routes"></a>
## 2. 选择连接方式

| 你的情况 | 选择 | 需要准备 |
| --- | --- | --- |
| 电脑在家庭、公司网络，无法开放公网入站端口 | Relay | 可访问的 Relay 域名、端口，以及为当前 Host 签发的登记令牌 |
| Host 有公网 IP，或可以映射公网端口 | 公网直连 | 本地监听 IP、对外业务入口和配对入口；放行两个 TCP 端口 |
| 已经配对，希望在同一局域网内使用 | 局域网 | Host 监听可达的内网地址；手机手动选择局域网路线 |

两种首次配对方式的手机操作相同：**扫描二维码 → 在电脑终端确认 → 自动连接**。二维码携带连接方式，手机不会再询问“直连还是 Relay”。

公网直连不要求 Relay 服务。未配置直连时，`agentpulse pair` 继续使用原有 Relay 流程；两者同时配置时，新二维码优先使用直连。直连失败不会偷偷转到 Relay。

局域网是既有设备的连接选择。默认情况下，只有同一个 Wi-Fi、但未配置 Relay 或显式直连入口，并不足以启动首次扫码配对。

<a id="installation"></a>
## 3. 安装电脑端与 Android App

### 3.1 获取源码

在希望保存源码的目录运行，将地址替换为项目总控仓库地址：

```bash
git clone --recurse-submodules <AGENTPULSE_REPOSITORY_URL> agentpulse
cd agentpulse
```

如果已经有总控仓库，在它的根目录执行：

```bash
git submodule update --init --recursive
```

后文“总控仓库根目录”指包含 `agentpulse-rs`、`agentpulse-android` 和本手册 `docs` 目录的位置。

### 3.2 从源码安装 Host

需要 Rust stable 工具链、Cargo，以及对应操作系统的本地编译工具。仓库使用 Rust 2024 edition；使用仓库的 `rust-toolchain.toml`。在总控仓库根目录运行：

```bash
cd agentpulse-rs
cargo install --path agentpulse-host --locked
agentpulse --version
agentpulse --help
```

安装后确认 Cargo 的可执行文件目录在 `PATH` 中。通常 Linux/macOS 是 `~/.cargo/bin`，Windows 是 `%USERPROFILE%\.cargo\bin`。

也可以只构建而不安装：

```bash
cargo build --release -p agentpulse-host --locked
```

产物为 Linux/macOS 的 `target/release/agentpulse`，或 Windows 的 `target\release\agentpulse.exe`。未加入 PATH 时，后文的 `agentpulse` 需要替换成这个文件的实际路径。

如果使用项目提供的构建包，应确认它与电脑系统、CPU 架构相符。本文不假定所有平台都有现成发布包；源码构建是可核对的安装路径。

### 3.3 安装 Android App

拿到适配当前 Host 的 APK 后，将 APK 传到手机，按 Android 的安装提示安装。如果系统询问是否允许当前文件管理器或浏览器安装应用，按提示设置。

需要自行构建时，在总控仓库的 `agentpulse-android` 目录准备：

- JDK 21，与当前仓库 CI 一致；
- Android SDK Platform 37 和 Build Tools 37.0.0；
- Android SDK 路径配置，例如 `ANDROID_HOME` 或本地 `local.properties` 中的 `sdk.dir`。

Linux/macOS：

```bash
./gradlew :app:assembleDebug
```

Windows PowerShell：

```powershell
.\gradlew.bat :app:assembleDebug
```

APK 位于 `app/build/outputs/apk/debug/app-debug.apk`。这个命令生成调试 APK，不等于签名发布版。发布签名说明见 [Android README](../agentpulse-android/README.md#signed-releases)。

使用公网直连时，Host 与 Android 都要包含直连实现；旧 App 不能识别新的直连二维码。

### 3.4 统一配置目录

正常使用默认配置目录即可。同一操作系统用户下的 `init`、`serve`、`pair`、`codex` 和 `stop` 应使用同一份配置。

需要隔离环境时，所有命令都加上同一个 `--data-dir`，例如：

```bash
agentpulse --data-dir /path/to/ap-config init --name "My Host"
agentpulse --data-dir /path/to/ap-config direct status
```

Windows 可以使用带引号的绝对路径，如 `"D:\AgentPulse\config"`。指定目录后不能在另一个终端遗漏这个参数，否则会访问另一套 Host 身份。`ap` 快捷脚本使用默认配置目录，没有直接透传 `--data-dir` 的选项。

<a id="relay-quickstart"></a>
## 4. 首次使用：已有 Relay

本节假定管理员已经提供 Relay 服务。如果还没有服务，先阅读[自建 Relay](#self-hosted-relay)。Relay 不是一个只填地址就能匿名使用的公共代理。

### 第一步：初始化电脑身份

```bash
agentpulse init --name "My Workstation"
```

记录输出的 `Host ID`，提供给 Relay 管理员，用于生成与这个 Host 匹配的登记配置和令牌。Host ID 与 Codex 的会话 ID 是两个不同的标识。

初始化只做一次。出现 `AlreadyInitialized` 时说明配置已经存在，不需要通过删除配置目录来重新开始。

### 第二步：配置 Relay

确保 Host 尚未运行；已经运行时先执行 `agentpulse stop`，使用 `ap` 启动的环境则先执行 `ap stop`。

Bash 中使用隐藏输入，避免把令牌直接写进命令历史：

```bash
read -r -s -p "Relay enrollment token: " relay_token
printf '\n'
printf '%s\n' "$relay_token" | agentpulse relay configure --endpoint relay.example.com:19191 --token-stdin
unset relay_token
agentpulse relay status
```

PowerShell：

```powershell
$relayToken = Read-Host "输入 Relay 登记令牌"
$relayToken | agentpulse relay configure --endpoint relay.example.com:19191 --token-stdin
Remove-Variable relayToken
agentpulse relay status
```

`--endpoint` 使用 `域名:端口`，不带 `https://`、路径或查询参数。当前 Relay 地址要求 DNS 名称，不能用 IP 字面量代替。登记令牌只给电脑端配置，手机从二维码获得自己的配对信息。

### 第三步：启动 Host，保持终端 A 打开

```bash
agentpulse serve --discover-threads --bind 127.0.0.1
```

`--discover-threads` 让 Host 跟随经由它启动或恢复的 Codex 会话，首次使用无需手工收集 Thread ID。

这里使用 `127.0.0.1` 是因为手机经 Relay 访问电脑，不需要直接进入电脑网卡。它要求 Relay 已经配置；这个监听方式不能直接供局域网手机连接。

### 第四步：在终端 B 打开二维码

```bash
agentpulse pair
```

等待二维码出现，然后按[扫码配对](#pairing)完成确认。Relay 模式会先等待临时配对路由就绪，所以不是执行命令后立即出码。

### 第五步：在项目目录打开终端 C，启动 Codex

```bash
agentpulse codex -- -C /path/to/project
```

Windows 示例：

```powershell
agentpulse codex -- -C "D:\work\my-project"
```

在这个 Codex 终端开始一段对话。手机连上后即可在会话列表中看到相应会话和后续事件。

不要用一个与 AgentPulse 无关的普通 Codex 进程来判断会话发现是否成功。通过 `agentpulse codex` 启动，才能连接到本次 Host 管理的 App Server。

<a id="direct-quickstart"></a>
## 5. 首次使用：公网直连

本节与 Relay 快速开始二选一。新环境先执行一次：

```bash
agentpulse init --name "Public Host"
```

### 5.1 区分本地监听与公网入口

| 配置 | 含义 |
| --- | --- |
| `--bind` | Host 本机网卡上实际存在的 IP，不带端口 |
| `--native-port` | 本机长期业务端口，默认 49320 |
| `--pairing-port` | 本机短时配对端口，默认 49321 |
| `--native-endpoint` | 手机访问业务连接的 `IP或域名:端口` |
| `--pairing-endpoint` | 手机扫码后访问配对服务的 `IP或域名:端口` |

必须区分这两组地址。云服务器显示的公网 IP 可能由平台映射到内网网卡，并不实际出现在本机网卡上。这时 `--bind` 填本机内网地址，对外入口填公网 IP 或解析到它的域名。

当前要求具体的本机单播 IP，不能用 `0.0.0.0`、`::` 或 loopback 作为直连配置的 `--bind`。业务和配对的本地端口必须不同。

### 5.2 情况 A：公网 IP 就在本机网卡上

下面的公网 IP 仅为示例，替换为实际地址：

```bash
agentpulse direct configure \
  --bind 203.0.113.10 \
  --native-endpoint 203.0.113.10:49320 \
  --pairing-endpoint 203.0.113.10:49321
```

在操作系统防火墙和云安全组中，允许手机访问 TCP 49320、49321。若改用了其他端口，则开放实际配置的端口。

### 5.3 情况 B：公网地址映射到内网网卡

例如本机网卡地址为 `192.168.1.20`，公网域名为 `host.example.com`，对外开放 44320、44321：

```bash
agentpulse direct configure \
  --bind 192.168.1.20 \
  --native-endpoint host.example.com:44320 \
  --pairing-endpoint host.example.com:44321
```

网络侧需要以下 TCP 映射：

| 手机访问的公网端口 | 转发到 Host |
| --- | --- |
| `host.example.com:44320` | `192.168.1.20:49320` |
| `host.example.com:44321` | `192.168.1.20:49321` |

域名要解析到正确公网地址。转发应保留 TCP 中的 Host TLS 连接，不要配置为终止 Host TLS 的普通 HTTPS 反向代理。AgentPulse 不自动创建映射或修改防火墙。

配对端口只在运行 `agentpulse pair` 时临时监听；平时探测这个端口不通是正常的。业务端口在 Host 运行期间持续监听。

### 5.4 Windows 与 IPv6 示例

PowerShell 可用一行完成配置，避免使用 Bash 的反斜杠续行：

```powershell
agentpulse direct configure --bind 192.168.1.20 --native-endpoint host.example.com:44320 --pairing-endpoint host.example.com:44321
```

IPv6 的入口地址必须带方括号，本机 `--bind` 不带方括号：

```bash
agentpulse direct configure --bind 2001:db8::10 --native-endpoint '[2001:db8::10]:49320' --pairing-endpoint '[2001:db8::10]:49321'
```

同样替换示例地址，并确认手机网络能到达该 IPv6 地址。当前不接受含接口 scope ID 的入口地址。

### 5.5 启动与配对

查看保存的配置：

```bash
agentpulse direct status
```

已有 Host 在运行时先停止它，再在终端 A 启动：

```bash
agentpulse serve --discover-threads
```

启用直连后，不必重复传入 `--bind`、`--port`。如果手动传入，它们必须与直连配置一致。不要沿用 Relay 示例中的 `--bind 127.0.0.1`。

终端 B：

```bash
agentpulse pair
```

手机扫描后自动选择直连，无需输入公网地址。配对成功后，手机保存的是长期业务入口，而不是二维码的短时配对端口。

终端 C：

```bash
agentpulse codex -- -C /path/to/project
```

配置保存后，后续日常命令不变。使用 `ap` 的 Linux 用户也仍然执行 `ap` 和 `ap qrcode`。

<a id="pairing"></a>
## 6. 扫码配对的完整过程

1. 确认 Host 已启动；在另一个终端执行 `agentpulse pair`。
2. 保持二维码完整可见。如果终端折行，扩大窗口或缩小字体后重新生成二维码。
3. 打开 Android App，选择“扫描二维码”或“开始扫描”，允许相机权限。
4. 将整个二维码放入取景范围。手机进入“正在等待电脑端确认…”时，查看生成二维码的那个终端。
5. 电脑会显示设备名称、设备 ID 和 `[y/N]`。核对设备后输入 `y` 并回车；直接回车或输入其他内容表示拒绝。
6. 手机显示“配对成功”，随后连接卡片应进入“已连接”。Relay 模式还会等待新设备的业务路由就绪。

每个二维码只有两分钟有效期，只能成功使用一次；配对会话最多允许五次尝试。过期、被拒绝、尝试过多或已经使用后，应重新执行 `agentpulse pair`，不要继续扫描旧截图。

二维码包含短时配对凭据，不应作为日常登录码长期保存。配对后通常无需重新扫码：手机保存设备凭据，下一次点击“连接”即可。

重新配对同一个 Host 会更新该手机的凭据。App 会先停止该 Host 的旧连接，成功后自动连接；如果重新配对失败，旧连接不会被无条件恢复，需要手动处理。

手机相机扫码是正式入口。当前没有手工输入配对 URI、蓝牙配对或将深链接作为首次配对入口的流程。

<a id="daily"></a>
## 7. 日常启动、退出与 Linux 快捷命令

### 7.1 所有平台可用的基本流程

- 终端 A：启动 `agentpulse serve --discover-threads`，根据所选路线添加必要的监听参数。
- 终端 B：首次配对或增加手机时运行 `agentpulse pair`。
- 项目终端：运行 `agentpulse codex -- -C <项目绝对路径>`。
- 查看状态：`agentpulse status`。
- 停止 Host：`agentpulse stop`。

`serve` 是前台进程，保持它所在的终端运行。关闭 Host 会停止其管理的运行时和 Codex App Server；进行中的任务会受影响。不要将“退出一个 Codex 终端界面”和“停止 Host”混为一谈。

指定不同的 Codex 可执行文件时，在启动 Host 时使用：

```bash
agentpulse serve --discover-threads --codex /path/to/codex
```

Relay loopback 环境同时加 `--bind 127.0.0.1`；直连环境由直连配置确定监听。Windows 应指向实际原生可执行文件，Host 不通过修改 PowerShell 执行策略来运行任意脚本包装器。

### 7.2 Linux 的 `ap` 快捷入口

`ap` 依赖 Bash、`flock`、systemd 用户服务和 `journalctl`，不作为 Windows/macOS 通用命令。Host 初始化和连接配置仍需先完成。

在 `agentpulse-rs` 目录安装脚本：

```bash
mkdir -p "$HOME/.local/bin"
install -m 755 scripts/ap "$HOME/.local/bin/ap"
export PATH="$HOME/.local/bin:$HOME/.cargo/bin:$PATH"
```

将 PATH 设置放到自己的 shell 配置后，新终端才能持续找到它。脚本默认从 PATH 查找 `agentpulse`，也可设置 `AGENTPULSE_BIN` 为其绝对路径。

| 命令 | 用途 |
| --- | --- |
| `ap` | 启动或复用 Host，然后在当前目录打开 Codex |
| `ap host` | 只启动或复用后台 Host，不打开 Codex，也不主动显示二维码 |
| `ap qrcode` | Host 运行时，显示新的单次配对二维码 |
| `ap status` | 查看 Host 状态 |
| `ap logs` | 查看最近日志 |
| `ap logs -f` | 持续查看日志 |
| `ap stop` | 停止后台 Host |
| `ap -C /path/to/project` | 指定 Codex 工作目录 |

新启动的 Host 会在普通 `ap` 流程中先显示二维码。已经配对、不想再次扫码时，可以按 `Ctrl+C` 跳过，再运行 `ap`；后台 Host 保留。

退出 Codex 终端界面后，后台 Host 仍可运行，手机仍能连接。系统注销、重启或用户服务生命周期另受本机 systemd 设置影响；脚本不是一个跨系统开机启动安装器。

`ap nona`、`ap rinia` 分别寻找 `codex-nona`、`codex-rinia` 可执行文件，它们是可选的本地包装器，不是项目自动创建的账户。没有这些包装器就使用 `ap`。

切换包装器，或改变当前 shell 的代理环境变量，可能使脚本重启 Host。Host 重启会重建本次运行的会话状态，所以不要在执行关键任务时无意切换环境。

<a id="phone"></a>
## 8. 手机上的会话、审批和消息

### 8.1 连接与会话列表

“我的连接”显示已配对电脑。选择“连接”后等待“已连接”，再进入会话列表。可以搜索会话，并按运行中、等待中或已结束等状态筛选。

当前 App 使用一个活动 Host 连接；保存多台电脑不代表同时实时连接所有电脑。没有会话时，先在电脑通过 `agentpulse codex` 启动对话。刚完成配对而尚未开始任何会话时，空列表是正常的。

手机窄屏使用列表与详情切换，宽屏可显示双栏。详情中可以查看消息、工具活动、计划、状态和待处理交互。

### 8.2 处理审批

会话出现“需要处理”时，打开审批卡片，查看操作、目录、网络目标或拟议文件修改，然后选择 Codex 提供的选项。

手机显示的选项来自请求本身，不保证每一种请求都只有“允许/拒绝”两个按钮。提交后等待结果；如果桌面已经处理或请求失效，旧卡片不能继续决定该操作。

出现“此请求只能由另一个 Codex 客户端处理”时，回到持有该请求的桌面客户端。手机端并不对所有桌面交互提供可写控制。

### 8.3 回答问题与 Plan 模式

结构化问题按字段顺序填写，一次提交整组答案。提供“其他”时可以输入自己的回答；敏感字段使用密码样式。提交后等待 Codex 处理。

输入 `/plan` 可选择启用或关闭 Plan 模式。正式计划完成后，按界面上的“实施计划”或继续修改选项推进。手机切换的是受管会话的 Plan 模式。

已知情况：部分已测 Codex 版本在手机选择实施后，桌面旧的计划确认框仍可能留在屏幕上。任务已开始时，用 `Escape` 关闭旧框，不要再重复确认。详情见 [Host 已知问题](../agentpulse-rs/agentpulse-host/README.md#known-issue-desktop-plan-confirmation)。

### 8.4 发送消息与队列

在会话输入框输入普通文字并发送。App 等待 Host 确认后再清空输入框；“等待 Host 确认”是传输确认，不表示每条消息都要有人在电脑前再次批准。

任务忙碌时，普通消息进入队列；希望引导当前正在运行的任务时使用 `/steer <内容>`。发送消息后先观察回执和事件，不要因为暂时没看到新回答而连续重复发送。

队列上限为每个会话 32 条、单条 64 KiB、所有会话合计 1 MiB。队列只在内存里存在，Host 重启后不会恢复。

### 8.5 后台与通知

连接期间 App 使用前台服务维持连接。允许通知后，可接收待处理交互、完成、失败或连接丢失等通知；普通事件不逐条推送。

断线后会在当前选择的路线上重试。点击“断开”会主动停止连接服务，不会自动取消电脑上的任务。系统强制停止 App 或机器网络不可达时，应重新打开 App 并连接；不能把前台服务等同于在任何系统省电策略下永远在线。

<a id="commands"></a>
## 9. 手机指令速查

以下命令输入在手机的会话消息框中，不是在电脑 shell 中执行。命令是否成功还取决于当前会话、Codex 能力和运行状态。

| 指令 | 用途 |
| --- | --- |
| `/model` | 获取可选模型并按界面选择 |
| `/model <模型ID> [推理强度]` | 选择模型；使用实际返回的模型与支持的强度 |
| `/resume` | 列出可恢复会话 |
| `/resume <会话ID>` | 恢复指定会话 |
| `/clear` 或 `/new` | 在当前工作区开始新会话，不是删除磁盘记录 |
| `/plan` | 显示启用/关闭 Plan 的操作 |
| `/plan off` | 关闭 Plan 模式 |
| `/compact` | 请求压缩当前会话上下文 |
| `/review [说明]` | 请求审查 |
| `/rename <新名称>` | 重命名会话 |
| `/fork` | 分支当前会话 |
| `/status` | 查看当前会话状态 |
| `/permissions` | 获取可选权限配置 |
| `/permissions <配置ID>` | 选择返回列表中的权限配置 |
| `/stop` | 取消当前任务，并暂停待发送队列 |
| `/queue pause` | 暂停队列 |
| `/queue resume` | 恢复队列 |
| `/queue clear` | 清空尚未发送的队列内容 |
| `/steer <内容>` | 向当前运行中的任务发送引导内容 |

手机 `/stop` 不会关闭 AgentPulse Host。停止电脑端服务使用终端命令 `agentpulse stop` 或 `ap stop`。

<a id="reconnect"></a>
## 10. 连接方式、配置修改与重连

### 10.1 配对后的路线

- 扫描 Relay 二维码后默认选择 Relay；扫描直连二维码后默认选择公网直连。
- 不会在连接失败时自动切换到其他路线。
- Relay 配对设备可以显式选择局域网，但 Host 必须监听实际可达的内网 IP。`--bind 127.0.0.1` 不满足这个条件。
- 公网直连设备保存独立业务入口，重试时不会用 mDNS 替换成局域网地址。

要让已经通过 Relay 配对的手机使用 LAN，可在停止 Host 后以 `--bind <本机内网IP>` 重新启动，并在手机选择“局域网”。确保防火墙允许 Native 业务端口，手机允许本地网络访问；mDNS 不可用时，地址发现可能失败。

### 10.2 修改直连入口

重新运行完整的 `agentpulse direct configure ...`，然后重启 Host。

- `agentpulse direct status` 显示磁盘上保存的配置。
- `agentpulse status` 显示当前进程实际使用的入口。
- 修改保存配置不会立即修改正在监听的端口，也不会更新已配对手机的地址。

对外地址或端口改变后，重新运行 `agentpulse pair`，让手机重新扫码获取新入口。仅修改本机映射、对外业务入口保持不变时，不必因为本地地址变化就更换手机保存的入口。

### 10.3 从直连恢复 Relay

先确认 Relay 已正确配置，再移除直连设置并重启。例如普通 CLI 环境：

```bash
agentpulse stop
agentpulse direct disable
agentpulse relay status
agentpulse serve --discover-threads --bind 127.0.0.1
```

在另一个终端运行 `agentpulse pair`，手机重新扫码后选择 Relay。原来保存的直连设备不会自动迁移。

如果 Relay 尚未配置，在启动 Host 前先按第 4 节配置。需要完全关闭 Relay 时，停止 Host 后运行 `agentpulse relay disable --confirm`；这会影响仍使用 Relay 的已有手机。

### 10.4 更新手机中的 Relay 地址

App 的“Relay 设置”可以保存地址，但不会替你部署 Relay、登记 Host 或配置电脑端令牌，也不会自动切换当前路线。换 Relay 服务后，通常应先完成电脑端配置，再重新扫码同步。

### 10.5 普通断线与进程重启

同一个 Host 运行期间的网络中断，可以按会话游标补齐缺失事件。Host 重启属于新的运行周期，App 会重置匹配的旧缓存。Android 进程退出也会丢失其内存缓存，重新连接后由当前 Host 提供可用状态。

不要把实时事件缓存当作跨 Host 重启的历史归档。`/resume` 可以请求恢复 Codex 提供的历史，但它与 AgentPulse 自己保存完整历史是不同的能力。

<a id="maintenance"></a>
## 11. 设备管理、数据保存与升级

### 11.1 查看和撤销手机

```bash
agentpulse devices list
agentpulse devices revoke <ANDROID_CLIENT_UUIDV7>
```

使用 `devices list` 输出的设备 ID。一个 Host 最多保存 16 个设备身份；设备不用了可以撤销后释放名额。撤销会使该设备的后续认证和活动连接失效，需要重新扫码才能再次使用。

手机“忘记”只删除手机本地凭据，不等于在 Host 上撤销设备。手机丢失时，应从电脑执行撤销。

### 11.2 重置全部设备凭据

需要撤销所有手机并更换 Host 的 CA/证书时，先停止 Host，再运行：

```bash
agentpulse credentials rotate --confirm-revoke-all
```

此操作保留 Host ID，但所有手机需要重新配对。它不是普通网络故障的第一步处理方式。

### 11.3 哪些数据会保存

| 内容 | 保存边界 |
| --- | --- |
| Host 身份、证书、设备凭据摘要、显式 Thread 列表 | 电脑配置目录 |
| Relay 和直连配置 | 电脑配置目录 |
| 手机已配对 Host 与设备凭据 | Android Keystore 支持的加密存储 |
| 本次运行的事件、交互和待发队列 | 内存；不保证跨进程重启保留 |
| Codex 自己的历史 | 由 Codex 管理，可通过其支持的恢复能力访问 |

默认配置目录由操作系统的应用目录规则决定。需要明确管理位置时使用统一的 `--data-dir`。配置里包含身份材料和 Relay 登记令牌，不要把整个目录贴进问题报告，也不要复制同一套 Host 身份到两台同时运行的机器。

Android 禁用了应用备份；更换手机、卸载或清除 App 数据后应准备重新配对。

### 11.4 升级

1. 等待关键任务结束，停止 Host。
2. 更新源码或使用匹配的平台构建包；总控仓库同时更新对应子模块。
3. 替换 Host 和 Android App；包含快捷脚本变更时也更新 `ap`。
4. 保留原配置目录，重新启动 Host，查看 `status`，再从手机连接。
5. 遇到协议不兼容时，先核对 Host/App 是否来自配套版本。

APK 覆盖安装要求签名兼容。如果系统提示签名冲突，不要误以为重新扫码可以解决安装问题；卸载会删除手机凭据，需要重新配对。直连功能使用新版二维码，旧 App 必须升级。发布版本是否包含本手册功能，应以该版本源码和变更说明为准。

<a id="self-hosted-relay"></a>
## 12. 自建 Relay

这是给服务器管理员的步骤。已有可用服务的普通用户只需要第 4 节。当前 Relay 配置绑定一个 Host ID，不是任意多个 Host 的开放注册服务。

### 12.1 准备服务器

需要一个手机和 Host 均可访问的公网服务器、解析到它的 DNS 名称，以及覆盖该名称的公开信任 TLS 证书链和对应私钥。开放选定的 TCP 端口，例如 19191。

Relay 使用自己的公网 TLS 证书；Host 的内部证书仍由 AgentPulse 管理。两者不要混用。Relay 接收外层连接并转发内部 Host TLS 密文，不读取会话或设备 Token 明文。

在服务器的 `agentpulse-rs` 源码目录构建：

```bash
cargo build --release -p agentpulse-relay --locked
```

将产物放到服务器可执行路径，以下命令假设可以直接运行 `agentpulse-relay`。先从 Host 初始化输出中取得真实 `Host ID`。

### 12.2 初始化与启动

替换路径、域名和 Host ID；由具有读取证书、写入配置目录权限的服务用户执行：

```bash
agentpulse-relay init \
  --config /path/to/relay.json \
  --bind 0.0.0.0:19191 \
  --public-endpoint relay.example.com:19191 \
  --certificate-chain /path/to/fullchain.pem \
  --private-key /path/to/privkey.pem \
  --host-id <HOST_UUIDV7>
```

命令只在初始化时输出一次登记令牌。将 `Host enrollment Token:` 后面的令牌值提供给该 Host 的管理员，不要把整段日志当成令牌，也不要直接将全部初始化输出管道传给 Host。

```bash
agentpulse-relay check-config --config /path/to/relay.json
agentpulse-relay serve --config /path/to/relay.json
```

另一个可访问服务器的终端：

```bash
agentpulse-relay probe --endpoint relay.example.com:19191
```

`probe` 成功表示公网 TLS 和 Relay 挑战服务可达，不代表 Host 已登记成功，也不代表手机已经配对。接下来仍要在 Host 按第 4 节配置令牌、启动和扫码。

### 12.3 长期运行与证书更新

仓库提供专用服务用户、systemd unit、受限部署和回滚脚本，详见 [Relay 部署说明](../agentpulse-rs/deploy/README.md)。其中已有环境的域名和路径仅用于解释该部署，不能直接视为你的可用公共服务。

证书续期后，更新证书文件，执行 `check-config`，重启 Relay，再执行公网 `probe`。当前部署不自动申请或续期证书。服务启动失败时查看服务日志和文件访问权限。

<a id="troubleshooting"></a>
## 13. 故障排查

先判断失败在哪一步：**Host 启动 → 出二维码 → 等待终端批准 → 配对成功 → 业务连接 → 会话发现**。这几步不是同一条连接，尤其直连有两个端口。

| 现象或错误 | 优先检查与处理 |
| --- | --- |
| 找不到 `agentpulse` / `codex` | 确认已安装、PATH 正确；非默认 Codex 用 `serve --codex` 指定 |
| `AlreadyInitialized` | 已有身份；继续使用现有配置，勿重复初始化 |
| `no Codex threads configured` | 首次使用选择 `serve --discover-threads`，或按高级用法登记 Thread ID |
| `loopback Native binding requires a configured Relay` | 未配 Relay 却监听 loopback；按所选路线配置，直连时去掉旧的 loopback 参数 |
| `serve --bind/--port conflict with direct configuration` | 参数与保存的直连设置冲突；直连模式省略重复参数，或统一配置 |
| `no private LAN address found` / 多个内网地址 | 非直连模式明确传 `--bind <内网IP>`；纯 Relay 可用 `127.0.0.1` |
| 本地端口被占用 | 检查另一份 Host、未结束的 `pair` 或其他服务；固定配对端口不会自动换号 |
| `QR pairing requires a configured Relay` | 当前运行的 Host 未启用直连，也未配 Relay；检查已安装版本、配置目录和重启情况 |
| 直连已 configure，但二维码仍是旧路线 | `direct status` 是保存值；重启 Host，再用 `status` 确认生效 |
| `could not publish QR pairing route` / 发布超时 | Relay 可达性、证书、Host ID 与登记令牌、Relay 容量；查看 `status` 和日志 |
| `Relay authentication_failed` | 地址、令牌与 Host 身份是否匹配；不要只修改手机 Relay 地址 |
| 手机无法识别二维码 | Host/App 版本匹配、二维码完整、相机权限、终端折行、光线与对焦 |
| `unsupported pairing URI/version` | 旧 App 扫描新版直连码，或版本不匹配；升级配套 App |
| 手机一直“等待电脑端确认” | 回到运行 `pair` 的终端，输入 `y` 并回车；不是在 `serve` 终端输入 |
| 配对过期、被拒绝或尝试次数耗尽 | 执行新的 `agentpulse pair` 并扫描新码 |
| 直连扫码连接超时 | 在 pair 运行时检查公网配对端口、DNS、云安全组、映射目标及本机监听 IP |
| 配对成功后业务连接超时 | 检查业务端口及映射；配对端口通不代表业务端口通 |
| 证书或指纹不匹配 | 使用新码，确认入口到达正确 Host、没有终止内部 TLS 的代理；不要关闭证书验证 |
| 选择局域网后连接失败 | Host 不能只监听 loopback；检查同网可达性、Native 端口、本地网络权限与 mDNS |
| 已连接但没有会话 | 用 `agentpulse codex` 或 `ap` 启动/恢复会话；普通独立 Codex 进程不自动被接管 |
| 输入框提示未知指令 | 对照第 9 节；手机输入框不是通用 shell，也不支持所有 Codex TUI 命令 |
| 断线后旧审批按钮不可用 | 请求可能已结束或被桌面处理；连接恢复后以当前待处理卡片为准 |
| Host 重启后看不到旧事件/队列 | 当前数据在内存中；按需使用 `/resume`，队列不会自动恢复 |
| 设备容量已满 | `devices list` 后撤销不再使用的设备，再扫码 |
| `ap` 启动失败 | Linux systemd 用户服务、`flock`、PATH、可执行包装器和 `ap logs`；其他平台使用基本 CLI |

常用诊断命令：

```bash
agentpulse --version
codex --version
agentpulse status
agentpulse direct status
agentpulse relay status
agentpulse devices list
```

Linux 使用 `ap` 时：

```bash
ap logs -f
```

检查监听端口可以使用 Linux 的 `ss -ltn`、macOS 的 `lsof -nP -iTCP -sTCP:LISTEN`，或 Windows PowerShell 的 `Get-NetTCPConnection -State Listen`。本机监听成功并不证明公网入站已经放行。

报告问题时提供 Host/App 版本、操作系统、所选路线、失败步骤和去除敏感内容的错误信息。不要附完整二维码、登记令牌、设备 Token、私钥或完整凭据文件。

<a id="reference"></a>
## 14. 电脑端命令速查与验收步骤

### 14.1 命令速查

| 命令 | 用途与前提 |
| --- | --- |
| `agentpulse init --name <名称>` | 初始化一次 Host 身份 |
| `agentpulse serve --discover-threads ...` | 启动 Host，跟随受管 App Server 会话 |
| `agentpulse codex -- -C <目录>` | Host 运行后，在指定项目打开 Codex |
| `agentpulse pair` | Host 运行后，生成一次配对二维码 |
| `agentpulse status` / `stop` | 查看运行状态 / 停止 Host |
| `agentpulse direct configure ...` | 保存直连设置，重启生效 |
| `agentpulse direct status` / `disable` | 查看 / 移除保存的直连配置；移除后也需重启 |
| `agentpulse relay configure --endpoint <域名:端口> --token-stdin` | 停止状态配置 Relay；从标准输入读取令牌 |
| `agentpulse relay status` | 查看已配置 Relay 地址 |
| `agentpulse relay disable --confirm` | 停止状态移除 Relay 配置 |
| `agentpulse devices list` / `revoke <设备ID>` | 查看 / 撤销已配对手机 |
| `agentpulse credentials rotate --confirm-revoke-all` | 停止状态轮换 CA 并撤销全部手机 |
| `agentpulse threads list` / `add <ID>...` / `remove <ID>...` | 管理显式 Codex 会话列表 |

每一级命令均可使用 `--help` 查看实际安装版本的参数，例如 `agentpulse direct configure --help`。

高级用法：如果只想加载指定会话，先 `threads add <CODEX_THREAD_UUIDV7>`，启动 `serve` 时不加 `--discover-threads`。列表保存后，在下次启动加载。`--discover-threads` 和 `ap` 不会把自动发现的全部会话永久写入这个列表；本地数据库也不是配置前提。

### 14.2 第一次连通后的检查

1. 手机连接卡片显示“已连接”。
2. 通过受管 Codex 新建或恢复会话，手机能看到会话与新消息。
3. 在手机发送一条普通消息，收到确认和后续响应。
4. 任务产生待处理交互时，在手机处理，电脑对应请求随之完成。
5. 手机断开后重新连接，不需要重新扫码；当前 Host 运行的状态可恢复。
6. 关闭并重新打开 App 后，用保存的 Host 再次连接。

### 14.3 验证确实没有使用 Relay

在测试环境先确保 Host 没有 Relay 配置，再配置公网直连。关闭手机 Wi-Fi，使用移动网络扫码并完成业务通信、断线重连和 App 重启恢复。

同时核对 `agentpulse status` 中的运行入口，以及服务器连接/网络记录中的实际目的地址和端口。分别验证公网网卡与端口映射部署；只在同一 Wi-Fi 下连接成功不能证明公网访问成立。

这一步属于部署验收。本手册不把尚未执行的公网实机测试记录为通过。

进一步阅读：[Host 说明](../agentpulse-rs/agentpulse-host/README.md)、[Android 说明](../agentpulse-android/README.md)、[Relay 部署](../agentpulse-rs/deploy/README.md)、[开发日志](../DEVELOPMENT_LOG.md)。
