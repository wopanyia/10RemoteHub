---
knowledge-role: reference
knowledge-status: current
---

# 遥控器接入：架构、阶段与能力边界（0.3.x）

本文回答三件事：0.3.0 的软件怎么组织遥控器和接收端，每个阶段能做到什么、做不到什么，以及新增一只遥控器时代码该接在哪里。操作引导见仓库根 `SKILL.md` 第 6 节，接入记录表见 [REMOTE-RECORD-TEMPLATE.md](REMOTE-RECORD-TEMPLATE.md)。流程图（本地网页）见 [REMOTE-ADAPTATION-FLOW.html](REMOTE-ADAPTATION-FLOW.html)。

0.3.0 只有一个软件入口（`python -m remotehub` / 安装包）：0.2.0 的固定型号主程序和 `--device-probe --learned-remotes` 开发工作台已经合并。正式安装包只发 macOS 和 Windows；Linux 代码继续保留在主干、暂不分发，Linux 上配套接收器仍走原有采集，共享 HID 读取端目前只有 macOS 和 Windows 实现。

## 1. 三个概念

| 概念 | 是什么 | 存什么 |
|---|---|---|
| **遥控器**（Remote） | 用户手里那只遥控器 | 名称、照片、键位与键码、配置方案（含按应用切换的按键层）、语音设置 |
| **接收端**（Connection） | 系统实际看到的输入来源：USB 接收器或蓝牙 HID 设备 | 连接方式、厂商/产品编号；可选限定的单元（序列号摘要） |
| **绑定规则** | 遥控器接受哪些接收端 | `connections` 列表；`units` 为空表示同类接收端插哪一个都接受 |

一只遥控器可以接受多个接收端（备用接收器、同时支持 USB 和蓝牙）；一个接收端也可以被多只遥控器接受（一个接收器同时能收几只遥控器）。电脑只看得到接收端，看不出是哪只遥控器按的键，能区分的只有键码本身。

运行规则：

1. 同一时间只启用一只遥控器的映射，用户在侧栏或「我的遥控器」里切换。
2. 启用的遥控器读取第一个在场、且被它接受的接收端；这个接收端断开后自动换到另一个在场的。
3. 共用接收端的两只遥控器，只按当前启用的那只来理解键码。添加时会提示“这个接收端也被「XX」使用”。
4. 遥控器麦克风跟着接收端走：只有专用采集的接收端和蓝牙棒直连能收音。

## 2. 读取路径与注册表选择

| | 专用采集（`receiver`） | 共享 HID（`hid`） | 蓝牙棒直连（`ble-dongle`）及其他注册读取端 |
|---|---|---|---|
| 适用 | 配套接收器 `USB 0416:0602` | 其他 USB／蓝牙 HID 按键接口 | 系统蓝牙读不了的 BLE HID 遥控器（如 Windows 上报 Code 10 的 `0416:0300`）；其他读取端按 `ReaderEntry.handles(conn)` 声明 |
| 读取方式 | Mac 后台读取组件独占读取；Windows WinUSB（Raw Input 降级） | Mac IOHID 普通打开（需输入监控）；Windows Raw Input，接收器 0416:0602 可走已配置的 WinUSB 按键接口 | 外接 USB 蓝牙棒一键切到 WinUSB，Bumble 用户态协议栈扫描、配对并读 GATT HID 报告（Windows 已实机验证；macOS / Linux 待验证） |
| 原按键 | 被接管，不会同时生效 | 日常使用：**Mac** 保持共享原始 HID 读取（不锁死空中鼠标、无需 root），通过 `hidutil` 按设备把已学按键映射到空键以屏蔽原按键，退出/切换/重连自动恢复或补设；**Windows** 通过独立线程的 Raw Input（按设备句柄识别来源）+ `WH_KEYBOARD_LL`（仅拦截目标遥控器已学键盘/媒体/音量键）协同屏蔽原按键（零驱动、退出即恢复、主键盘不受影响；受 Windows UIPI 隔离，普通权限不拦截管理员权限窗口），通用 USB 接收器亦可在添加窗口一键让 App 接管（系统自带 WinUSB），实现驱动级独占。失败时均退回共享读取并在状态里说明。添加/学键窗口一律共享读取、不屏蔽原按键 | 蓝牙棒直连时遥控器只连 App，原按键不再进入系统 |
| 语音 | 遥控器麦克风（采集 → G.722.1 解码 → 虚拟麦克风）或仅触发识别 | 仅触发识别快捷键；麦克风由识别软件选择 | 命中 `FIBER0D`（G.722.1）或 `ATV Voice`（IMA-ADPCM）时用遥控器麦克风（实时解码 → 虚拟麦克风），否则仅触发识别；由注册项 `remote_mic` / `remote_mic_for` 声明 |
| 启动 | USB 清单发现接收器后自动接入（`ensure_mapping`） | 每 2 秒扫描一次已接受的接收端（`scan_hid`） | 非独占读取端统一由 `scan_hid` 每 2 秒扫描并在 `CompositeReader` 中合并进添加窗口 |

`controller_remotes.RemoteInputMixin.input_path()` 不再写死二选一，而是委托 `readers.ReaderRegistry.select()` 按当前遥控器支持的注册项优先级与在场状态选出唯一路径（例如同时配了 `0416:0602` 与另一种连接时，`0416:0602` 在场优先走 `receiver`，不在场时自动回落到另一个已注册读取端）。任意时刻仍只允许一条读取路径运行（Windows Raw Input 每个进程只能注册一个接收者），切换遥控器、切换读取端或打开添加窗口时会先通过 `pause_remote_input` / `_stop_readers` 停掉其余读取端，结束后再恢复（`resume_remote_input`）。

## 3. 代码位置

```
readers.py                            KeyReader 协议、ReceiverReader、CompositeReader、ReaderEntry 与 ReaderRegistry
remote_library.py                     Remote / Connection / RemoteLibraryStore（remotes.json，版本 2）
                                       首次读取时迁移 0.2.0 profiles.json 与开发期 learned-remotes.json（只读不改）
controller.py                         AppController：组装状态、信号与基础按键学习/分发，并组合下列 4 个职责 mixin
  ├─ controller_remotes.py            RemoteInputMixin：遥控器库操作、注册表选路、读取端互斥启停、专用接收端状态
  ├─ controller_voice.py              VoiceControllerMixin：语音设置、虚拟麦克风、USB 语音链路与 WAV 测试
  ├─ controller_overlays.py           OverlayMacroMixin：九键/九宫格/速查浮层、宏动作、切换器与导航角色映射
  └─ controller_profiles.py           ProfileLayerMixin：配置方案切换/导入导出、前台应用自动切层与覆盖
learned_remotes.py                    LearnedRemoteRuntime（通用共享/已注册读取端运行时）；LearnedRemoteStore 仅用于迁移
mac_hid_probe.py / windows_probe.py   共享 HID 读取端
devices.py / usb_bridge.py ...        专用采集与遥控器音频（0.2.0 原样保留）
device_probe.DeviceProbeDialog        添加遥控器 / 照片学习对话框
ui/remote_enrollment.py               在照片学习对话框里命名并保存遥控器
ui/remote_list.py                     「我的遥控器」页：已添加的遥控器 +「能接入哪些遥控器」（接入档位与麦克风档位）
ui/sidebar.py                         当前遥控器、连接状态、添加遥控器、学习更多按键
ui/onboarding.py                      首次引导：添加遥控器 → 系统准备（按接收端类型）→ 验证按键
ui/enrollment_voice.py                「仅触发识别」的试用卡片：触发键、听写确认与诊断（麦克风只在语音页第 1 步选）
models.py + models/*.json             内置接收器身份（主窗口检测提示、0.2.0 固定型号迁移用的照片键位）
protocols.py                          协议库：按连接识别按键协议与语音协议（`Remote.voice_protocol` 的来源）；对用户只说接入档位与麦克风档位（`access_heading`、`mic_text`）
key_roles.py                          统一按键角色（方向区 6 键 / 数字区 12 键 / 语音 1 键，功能区留空）与标准 HID 键码反推
enrollment.py                         添加流程各步提示与脱敏诊断
```

### 读取端约定（新增读取方式时实现）

| 方法 / 属性 | 约定 |
|---|---|
| `enumerate() -> list[ProbeDevice]` | 列出可读设备。`ProbeDevice`：`id`（会话内唯一）、`name`、`transport`（`"USB"` 或 `"蓝牙"`）、`vendor`、`product`、`unit_key`（稳定序列号的 sha256 十六进制，无则空）、`vendor_id_source`、`bcd_device` |
| `peek() -> list[ProbeDevice]`（可选） | 只读快照列举，不调用 `stop()`、不打断正在进行的监听 |
| `start(identity, on_key, on_removed, *, expected=None) -> (opened, total)` | 只打开所选设备。`on_key(channel, trigger, down)` 每个按下/松开各调用一次；拔出调用 `on_removed()`；`expected` 与当前枚举不一致时抛 `RuntimeError` |
| `stop()` / `close()` | 停止回调、释放接口 |
| `scope`（可选） | 当前读取范围的一句说明 |

`trigger` 只能是 `hid:7:<usage>`、`hid:12:<usage>` 或 `win:<scan>:<flags>`，规则在 `remote_library.TRIGGER`（`enrollment.py` 和 `learned_remotes.py` 也有同样的校验，修改时同步）。

#### 新增一种读取方式要改哪些地方

1. **新建读取端模块**：实现上述 `KeyReader` 接口（`enumerate`、`start(identity, on_key, on_removed, *, expected=None)`、`stop` / `close`，可选 `peek` 和 `scope`），输出标准 `trigger`。
2. **注册 `ReaderEntry`**：在 `desktop/remotehub/readers.py` 的 `_DEFAULT_REGISTRY` 注册一条 `ReaderEntry`，声明 `handles(connection)`（能处理哪些接收端）、`factory`、`remote_mic` / `remote_mic_for`（是否提供遥控器麦克风）、`dedicated`、`priority`（专用采集 `10`，特定扩展建议 `50`，通用 HID 兜底 `200`）和 `platforms`。
3. **不修改控制器**：`AppController` / `RemoteInputMixin`、`DeviceProbeDialog`（通过 `CompositeReader`）、`Remote.supports_remote_mic` 均从注册表读取能力与选路，**不需要改 `controller.py` 或 `input_path()` 的分支代码**。
4. **按需扩展边界校验 / 私有音频**：若使用新的 `trigger` 前缀（非 `hid:7:` / `hid:12:` / `win:`），同步更新 `remote_library.TRIGGER` 与前述两处校验；若 `remote_mic=True` 且使用私有音频协议，还需补充对应音频桥接并在实机完成 6 步语音验收。
5. **补充测试**：参考 `desktop/tests/test_reader_registry.py`，使用假读取端注册并验证选路、键值分发、状态文案、麦克风能力声明与路径切换互斥。

## 4. 添加一只遥控器：一步一步到能用

| 步骤 | 界面 | 软件做什么 | 通过标准 |
|---|---|---|---|
| 1 连接 | 插接收器或系统设置里配对蓝牙 | 不参与配对 | 系统里能看到 |
| 2 添加 | 「添加遥控器」→ 选中设备 →「开始检测」 | 暂停当前映射，只监听所选设备 | 收到一对按下和松开 |
| 3 照片学键 | 添加窗口显示接入档位（✅ 直接接入 / 🔧 点一下就能接入 / ⚠️ 目前无法自动接入 / ❓ 看不到设备）→「导入照片，开始学习」→ 点四角摆正 → 回答「有没有麦克风」，有就说明麦克风档位（直接能用 / 换蓝牙棒直连能用 / 目前用不了）并先在照片上点语音键位置 → 逐个点选按键位置 → 右侧两步选「分区（方向区 / 数字区 / 功能区）→ 键位 / 自定义名称」（支持先选后按实体键，也支持直接按标准 HID 键自动识别分区、角色或常见媒体键名）→ 保存 | 每个键记录键码、位置与可选 `role`（19 个核心角色，功能区留空自定义）。语音键：命中内置语音协议（`0416:0602`、`FIBER0D`、`ATV Voice`）不按语音键，保存为 `voice_protocol`，`0416:0602` 自动绑定接收器语音键码并锁定为开始/结束；未命中时请用户按一下语音键记下键码。学键期间点错的位置可随时删除；无 12 秒空闲超时，仅保留 4 秒未收到松开自愈；没有数字区的小遥控器跳过九键即可 | 至少一个普通键 |
| 4 保存 | 自动 | 新建遥控器，接受“同类接收端的任意一个”，切到它，恢复读取 | 侧栏显示“已连接” |
| 5 系统准备 | 首次引导第二步 / 通用设置 | 保存后自动弹出「还差几步」清单（必须 / 可选），侧栏常驻「完成设置（还差 N 项）」。配套接收器：Mac 读取组件 + 辅助功能，Windows 在添加窗口一键让 App 接管（Zadig 备选）；系统蓝牙读不了的 BLE 遥控器：外接蓝牙棒切换直连；其他：Mac 输入监控 + 辅助功能；用遥控器麦克风时可选装虚拟麦克风 | 权限显示已授权 |
| 6 动作 | 照片上点键 → 选动作 → 按实体键 | 执行动作 | 用户确认动作完成 |
| 7 语音 | 「语音设置」 | 第 1 步「语音来源」是唯一的麦克风选择：有 `voice_protocol` 的遥控器可选遥控器麦克风；选「仅触发识别」时下方出现试用卡片：指定触发键 → 触发识别 → 用户确认已听写 | 识别软件出字 |
| 8 学更多 | 侧栏「学习更多按键」 | 学到的键同步到这只遥控器的所有配置方案；从新类型接收端学习时自动接受该类型 | — |

### 照片学习的流程（3b 细节）

1. **导入与摆正**：选照片后点遥控器四个角（鼠标移入照片区域即显示手指光标、角点上显示张开手、拖动时显示握拳），`photo_cutout.rectify_photo` 把整张照片拉平裁好（也可「使用原图」）。
2. **逐个点选与两步分类**：导入完成后照片保持干净（不再自动画框，避免多键遥控器像素误认）。点击照片上的一个按键位置，下方显示两步分类卡片：
   - **第 1 步选分区**（附示例）：`方向区`（上下左右/确定/返回）、`数字区`（0–9 / `*` / `#`）、`语音区`（语音键）、`功能区`（音量/主页/自定义）；
   - **第 2 步选键位或自定义名称**：方向区 6 键与数字区 12 键直接点选角色按钮（已绑定的角色带 `✓` 标记，重选自动保持唯一性）；功能区按下实体键后可直接在下方输入自定义名称。
   - **两种顺序均支持**：既可以先点照片 → 选分区与键位 → 按实体键（适用于厂商非标准键码，按完后锁定用户所选分区与角色），也可以点照片后直接按实体键（标准 HID 方向/数字/返回键自动勾选对应分区与键位，常见媒体键自动预填中文名并归入功能区）。
3. **稳定等待与微调**：学键等待期间不设 12 秒空闲超时，只保留按下后的 4 秒未收到松开自愈保护；已学或待学的高亮框均可拖动位置、点击 `＋`/`−` 缩放大小，或直接改名/删除。
4. **保存**：只要已学至少 1 个键即可直接点「保存遥控器」（保存时自动停止当前等待）；更换照片时会保留已学按键供核对位置。

同一颗键在不同接收端发不同的码时，在「学习更多按键」里试按新接收端，按提示归到同一颗键（`extra_triggers`）。

## 5. 各阶段能力边界

| 阶段 | 做不到的事 |
|---|---|
| 系统连接 | 软件不做配对；系统看不到、或连接反复断开时无法处理 |
| App 能看到 | Mac 只列出报告描述含键盘页 7 或消费者页 12 的设备；Windows 只列 Raw Input 键盘和消费者控制集合。页 9 按钮、厂商私有页、系统控制页、私有 USB 接口、只有 GATT 服务的蓝牙都看不到 |
| 试按 | 两颗实体键报告完全相同时只能当作同一个键；添加窗口共享读取，不拦截原键 |
| 多接收端 | 当前同一时间只读取一个在场的接收端（读取端一次只打开一个设备）；同时插两个时，另一个在前一个断开后才接上 |
| 共用接收端 | 无法区分是哪只遥控器按的；按当前启用的遥控器理解键码 |
| 动作 | App 发出按键不等于目标软件收到；Windows 管理员窗口、豆包输入法等限制同 0.2.0 |
| 语音·触发识别 | 只能确认“快捷键已发出”，出字由用户确认 |
| 语音·遥控器麦克风 | 共享 HID 路径没有。内置私有收音只有配套接收器 `0416:0602` 和蓝牙棒直连时的 `FIBER0D` / `ATV Voice`。系统本身把遥控器麦克风暴露为标准输入时，可在识别软件里直接选它；其他私有音频协议需要写适配器 |

### 私有音频适配器的分段验收

1. 录音开始/结束控制；2. 音频报文确实到达；3. 分片重组与格式；4. 解码为可听的 PCM；5. 送入识别软件；6. 断开重连后仍能录音。离线回放通过只说明第 1～4 步成立。

## 6. 已踩过的坑（新设备照此先查）

| 坑 | 出处 | 应对 |
|---|---|---|
| 系统已配对 ≠ App 能读到按键 | BLE 遥控器 FIBER0D：Windows 配对后系统不暴露 HID 接口 | 系统连接与 App 可见分开记录；不暴露 HID 时只能另做 GATT 读取端 |
| 只收到按下不算学会 | 通用试按 | 必须成对；缺松开时换一颗键 |
| 遥控器键和主键盘同名键混淆 | 蓝牙键盘 620A:0407 ↑ | 只监听所选设备，再用主键盘同一颗键做对照 |
| 某颗键在普通键盘路径丢失 | 0416:0602 的 `#` 键在 Windows Raw Input 下缺失 | 完整键位走 WinUSB，记为平台限制 |
| 接口类型与传输类型判断混淆 | 0416:0602复合 USB | interface class 与端点传输类型分开判断；截断抓包不补零 |
| 一只遥控器多个接收器 / 一个接收器多只遥控器 | 0416:0602同类接收器 | 遥控器与接收端分开建模，默认接受同类接收端的任意一个 |
| 松手后还会到最后一帧音频 | BLE 遥控器 FIBER0D | 停止命令与音频结束分开处理 |
| 收到音频 ≠ 能解码 ≠ 识别出字 | 0416:0602、FIBER0D | 三项分别验收 |
| 换 USB 口后 Windows 复用设备路径 | Windows 枚举 | 用实时句柄判断选择是否失效 |
| Raw Input 每进程一个接收者 | Windows 统一入口 | 同一时间只运行一条读取路径 |
| 默认会改变系统状态的操作 | 全部 | 不默认 detach/reset/reconfigure USB；只在明确授权后做最小采集 |

## 7. 待办与已知限制

1. 读取端同时打开多个接收端（Mac 共享 IOHID、Windows Raw Input 都可扩展）。
2. 扩展读取端实现：新的厂商私有协议或 Linux 共享 HID 后续直接实现 `readers.KeyReader` 并在 `readers.py` 新增一条 `ReaderEntry` 注册项即可接入（可参考已合入并实机验证的 `ble_dongle_reader.py`），无需修改 `AppController`（注册表机制与互斥选路已就绪）。
3. 按收到键码的接收端自动切换遥控器（共用接收端时仍按当前启用的遥控器）。
4. 页 9 按钮与系统控制页的支持需要同时放宽枚举过滤和键码校验。
