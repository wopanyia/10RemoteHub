# Windows 端说明

本文记录 Windows 实现的结构、0.2.0 专项优化、0.3.0 / 0.3.1 的一键接管与蓝牙棒直连、依赖、验收结果和已知限制。安装步骤见 [INSTALL-Windows.md](../INSTALL-Windows.md)。

## 1. 架构

Windows 与 macOS / Linux 采用同一个模型：**采集程序独占接收器，主程序只处理业务**。

```text
遥控器
  │ 2.4G
接收器（USB 0416:0602）
  ├─ 接口 1 消费者控制 ─┐
  ├─ 接口 2 鼠标       ├─ windows_collector.py：WinUSB 独占，读原始 HID 报文
  ├─ 接口 3 厂商语音   │   用 HIDReports 解析成固定键表的 hid:* 标识
  └─ 接口 4 键盘      ─┘
        │
        ├─ 按键事件 → controller.on_key → 动作（VK 组合键 / 鼠标 / UIA 控件与菜单 / 浮层 / 动作链…）
        └─ 语音会话 → G.722.1 解码 → AudioBridge → CABLE Input → 识别软件
```

核心要点与 `0.2.0` 专项优化：

- **进程内 `<1 ms` SetupAPI 枚举与双接口状态诊断**：`inventory.py` 与 `windows_driver.py` 直接在进程内通过 `setupapi.dll`（`SetupDiGetClassDevsW` + `DEVPKEY_Device_Service`）枚举 `USB\VID_0416&PID_0602` 及 `MI_03` / `MI_04` 服务绑定状态，完全取代旧版每 2 秒拉起 `powershell.exe` 子进程的做法（消除后台 CPU 抖动与控制台开销），并在通用设置分别展示 Interface 3 与 Interface 4 绑定状态。
- **按键来源于原始报文，不是 Raw Input**：Windows 键盘驱动会直接丢弃遥控器 `#` 键（HID 用法 `0xCC`），且 Raw Input 无法抑制物理按键。`windows_collector.py` 独占接口 1/2/3/4 并补齐与 macOS 一致的厂商语音帧（`01 82 03` 按下请求、`01 86 00` 松开结束）；Raw Input 仅在接口 4 未绑定时作为降级路径（失败停止时正确发出 `capture_stopped` 状态信号）。
- **与输入法/Shift 无关的虚拟键码注入与原生滚轮/锁屏**：`actions.py` 在 Windows 下优先按物理虚拟键码（`VK_A`–`VK_Z`、`VK_0`–`VK_9`、OEM 符号键）投递 `KeyCode.from_vk(...)`，避免 `Shift` 组合键或中文输入法激活时字符映射失效；补齐 `KeyOutput.scroll()` 滚轮支持（及 PyInstaller `pynput.mouse._win32` 隐藏导入）；锁屏直接调用 `user32.LockWorkStation()`（消除模拟 `Win+L` 导致的 `Win` 修饰键残留卡死）。
- **纯 `ctypes` Windows UI Automation (`win_uia.py`) 与零轮询前台跟踪**：
  - 无额外第三方依赖，封装 COM `CUIAutomation`（MTA 工作线程）实现 **聊天输入框自动聚焦**（`chat_focus` + `voice_auto_focus`）、**界面控件悬停捕获与点击/聚焦**（`ui_element`）以及 **应用菜单命令**（`app_menu`，同时兼容传统 Win32 `GetMenu` 与现代 UIA `MenuBar`）；
  - 使用 `SetWinEventHook(EVENT_SYSTEM_FOREGROUND)` 零轮询监听前台窗口切换，支持解析 UWP `ApplicationFrameHost.exe` 内部的真实 `CoreWindow` 进程、读取 `version.dll` `FileDescription` 应用友好名称，以及纯二进制解析开始菜单 `.lnk` 快捷方式；
  - 检测前台窗口管理员提权状态（UIPI `TokenElevation`），受阻时给出明确诊断提示。
- **非抢焦点屏幕浮层与多显示器 DPI**：启用 Per-Monitor V2 DPI 感知；`ui/osd.py` 为速查浮层、九宫格与方案切换提示应用 Win32 `WS_EX_NOACTIVATE | WS_EX_TOOLWINDOW | WS_EX_TOPMOST`（纯提示浮层叠加 `WS_EX_TRANSPARENT` 鼠标穿透）与 `SetWindowPos(..., SWP_NOACTIVATE)`，确保弹出与点击不抢夺当前前台窗口焦点。

### `0.3.0` 新增

- **一键接管 USB 接收器（`win_usb_driver.py`）**：不再需要 Zadig。App 以管理员身份（UAC `runas`）启动自身的提权子进程，用 SetupAPI `DiInstallDevice`（`DIOD_INHERIT_CLASSDRVS`）把选中接口绑定到系统自带、微软签名的 `%SystemRoot%\INF\winusb.inf`，写入固定的 `DeviceInterfaceGUIDs` 让 libusb 立即可用，再重启设备节点。`takeover_interface` 按接口用途挑选：键盘、媒体键与厂商数据接口接管，含指针的接口、标准声卡和存储不碰；`0416:0602` 只接管 `MI_03` / `MI_04`。电脑上没有其他键盘时拒绝；非 `0416:0602` 接管后 15 秒内未点「保留接管」，已提权的子进程自行恢复，不需要第二次授权。恢复时清掉写入的注册表值，删除属于该设备的旧 Zadig / libwdi `oemNN.inf`，移除设备节点并 `pnputil /scan-devices`，Windows 自动装回默认 `HidUsb` / `BTHUSB`。主板自带蓝牙一律拒绝切换。
- **外接 USB 蓝牙棒直连（`ble_dongle_reader.py`，依赖 Bumble）**：Windows 蓝牙栈拒载的 BLE HOGP 遥控器（`BTHLEDEVICE\{00001812…}` 报 Code 10）由 `WindowsProbeMetadata.blocked_ble_remotes()` 列出，并从设备实例里取出蓝牙地址（只用于匹配，不显示）。外接蓝牙棒一键切换到 WinUSB 后，Bumble 作为用户态蓝牙协议栈直接扫描、配对（绑定密钥存在 `ble-dongle/keys.json`）、读取 GATT HID 报告与语音特征；需要固件的蓝牙棒（如 `0BDA` 系列）会自动从 Windows DriverStore 取得固件。
  - 扫描：选中系统列出的遥控器时只找它（按地址，其次按名字），找到即停止并连接；未选中时按广播判断把握，带 `0x1812` 服务或名字像遥控器的才自动连接并提前结束扫描，只靠外观类型等弱特征匹配的设备由用户确认。连上后没有 HID 服务的设备从列表和 `keys.json` 中删除，本次运行内不再提供。
  - 语音：`FIBER0D` 走 G.722.1、`ATV Voice` 走 IMA-ADPCM，解码后经 `VoiceSession` → `AudioBridge` → `CABLE Input`。Bumble 默认 15–30 ms 连接间隔且接受遥控器提出的任何更慢间隔，会让语音落后说话数秒；现在连接时请求 7.5–15 ms，每次语音开始时若链路被放慢再请求一次。
  - 日志 `%LOCALAPPDATA%\RemoteHub\ble-dongle\session.log`：扫描结果（含每个设备的服务与外观）、连接间隔及其变化、每次说话的送达时序（`first_after`、`delivered` 与 `audio`、`after_release`）。
- **通用 HID 遥控器按设备屏蔽原按键（`win_hid_suppress.py`）**：专用线程同时持有 Raw Input sink 与 `WH_KEYBOARD_LL`，只吞掉来自所选遥控器句柄的已学按键，内建键盘与未知来源一律放行；管理员前台窗口受 UIPI 限制不拦截。

### 语音协议（与 macOS / Linux 采集一致）

| 方向 | 端点 | 内容 |
|---|---|---|
| 读 | `0x84` 中断 IN | `01 82 03` / `01 86 …` 语音键帧；`01 81 …` 音频帧；`01 08 01 00` / `01 01 01 00` 握手；`01 ff` 停止确认 |
| 写 | `0x05` bulk OUT | `01 08 01 00` 就绪；`01 82 01 00` + `01 08 01 00` 开始；`01 01 01 00` 进入音频；`0a 54 45` 每秒保活；`01 00` 停止 |

### 虚拟麦克风与多声卡路由

Windows 依赖外部 **VB-CABLE** 虚拟声卡：

- 应用把解码后的 PCM 播放到选定的 `CABLE Input`（多条虚拟线时可在语音设置下拉切换）；
- 「录音时切换系统默认麦克风」模式下，`WindowsInputRoute` 通过 `IPolicyConfig` 在录音期间将系统默认录音端（console / multimedia / communications 三个角色）临时切换到同组配对的 `CABLE Output`，结束或异常退出时恢复原设备。

## 2. 依赖与绑定

| 组件 | 作用 | 是否随项目分发 |
|---|---|---|
| Windows 自带 `winusb.inf` | App 一键接管接收器 / 切换蓝牙棒 | 系统自带 |
| Zadig | 一键接管失败时的手动备选（仅 `0416:0602`） | 否，用户自行下载 |
| Bumble 及其依赖 | 外接蓝牙棒直连的用户态蓝牙协议栈 | 是（Python 依赖，Apache-2.0 等） |
| VB-CABLE | 虚拟声卡 | 否，用户自行安装 |
| libusb / pyusb | 访问 WinUSB 接口 | 是（Python 依赖） |
| PySide6 | 界面与音频输出 | 是 |
| Win32 / COM UIAutomation | 窗口、快捷键、聊天聚焦、控件与菜单自动化 | 系统自带（`ctypes` 直接调用） |

## 3. 验收记录

### 2026-11-22 `0.3.1` 复合 USB 接收器一键接管与清单闭环实机

验证环境：Windows 11 x64，配套 `0416:0602` 复合 USB 接收器（`MI_03` / `MI_04`）。

- 修复 `_list_present_usb_nodes_windows` 误拒 `&MI_XX` 复合子接口导致看不到一键接管按钮的问题，合并子接口 `HardwareIds` 与 `CompatibleIds`，并对已知专用接收器 `0416:0602` 放行单键盘拦截。
- 修复「完成设置」清单点击接管跳回添加向导的内循环，支持在「添加遥控器」、「完成设置」清单与「通用设置 → 设备连接」直接一键接管并变为「已接管」；实机测试确认接管成功。
- 原生全量自动化 **919 passed / 62 skipped**，二进制与 CLI 隐私门禁 0 命中。

### 2026-11-21 `0.3.0` 蓝牙棒直连实机

验证环境：Windows 11 x64，主板蓝牙 + 外接蓝牙棒 `0BDA:8771`（WinUSB），遥控器 `0416:0300`（固件 `FIBER0D_U_R01_592_34516`，系统蓝牙下 Code 10）。

- 选中系统列出的遥控器后扫描，只连接该遥控器；附近的运动相机不再被连接。
- 按键与遥控器麦克风可用；四次说话的送达时序为 `delivered` ≈ `audio`（如 3.78 s / 3.76 s）、`after_release=0.00s`，连接间隔保持 10–15 ms。修复前同一遥控器最后一帧在松开语音键后约 3 s 才到达。
- 用户反馈：可用，但识别体验仍不如 macOS 上的 USB 遥控器，原因未定位。
- 原生全量自动化 **917 passed / 62 skipped**，`release_policy.py` 门禁与许可收集通过。

### 2026-11-18 `0.2.0` 原生构建与门禁验证

验证环境：Windows 11 x64（`10.0.26200`），Python 3.12.10。

- 完成 SetupAPI 进程内枚举、双接口（`Interface 3` / `Interface 4`）状态胶囊、VK 物理键码投递、`LockWorkStation()`、Win32 非激活浮层（`WS_EX_NOACTIVATE` + `WS_EX_TRANSPARENT`）、Windows UIA 聊天聚焦 / 控件悬停捕获（含自动识别任意前台 `.exe` 模式、Electron `Chrome_RenderWidgetHostHWND` 唤醒与多进程/WebView2 归属匹配）/ 菜单动作、官网 `.exe` 与 MSIX 自动发现及 9 项 Windows 专属常见动作优化；
- Windows 11 原生全量自动化回归 **642 passed / 63 skipped**、离屏 `--smoke` 通过；
- PyInstaller 原生打包生成 `10RemoteHub-0.2.0-win-x64.zip`，通过 `release_policy.py` 门禁、651 个文件 + 756 个冻结模块字节码零个人标识隐私扫描、干净目录解包启动与 `--diagnose` 脱敏核验，原有用户配置保持不变。

### 2026-11-12 发布文件

Windows 已实机验证基础链路。原生自动化 442 passed / 57 skipped；ZIP 在独立目录通过解压启动、旧版替换、回滚、卸载和配置保留，诊断报告完成脱敏核对。

### 2026-11-09 端到端实机验证

验证环境：Windows x64，Python 3.12.10。接口 1/2/3/4 独占、`#` = `hid:7:204`、语音键按下/松开、16 kHz 单声道回录峰值 RMS 0.30、微信输入法端到端出字全部通过。

## 4. 已知限制

1. **豆包输入法无法唤醒**：过滤程序模拟按键；微信输入法可用。
2. **管理员权限窗口（UIPI）**：普通权限进程无法向管理员窗口注入输入或读取可写 UIA 焦点。
3. **Windows 语音输入（Win+H）**：能唤醒但识别不到虚拟声卡音频。
4. 应用未运行时遥控器不产生按键（WinUSB 完整重映射预期行为）。

## 5. 排障

| 现象 | 检查 |
|---|---|
| 左侧不显示 `USB 0416:0602` | 接收器是否插好；`10RemoteHub.exe --diagnose report.json` 的 `usb` 字段 |
| 按键无反应或 `#` 键无效 | 通用设置查看 `USB 0416:0602` 是否显示「已接管」（未接管或部分接管时直接点「让 App 接管这个接收器」）；前台是否为管理员权限窗口（查看 UIPI 提示） |
| 语音不可用或灯常亮 | 通用设置查看 `USB 0416:0602` 是否显示「已接管」（含 `Interface 3`）；检查 `connection.log` 的 `request stop` 与 `01ff` |
| 输入法有电平但不出字 | 识别快捷键是否与输入法一致（微信输入法为 `Alt`） |
| 输入法没有电平 | 连接方式选「录音时切换系统默认麦克风」并把输入法设为系统默认；多虚拟声卡时核对选中的 `CABLE Input` |

## 6. 构建与调试

```powershell
cd <项目>\desktop
powershell -ExecutionPolicy Bypass -File scripts\build_windows.ps1 -LicenseSupplement <材料清单绝对路径>
```

国内网络直连 PyPI 失败时脚本会自动回退镜像，也可显式传 `-IndexUrl https://pypi.tuna.tsinghua.edu.cn/simple`。PyInstaller 不能交叉编译，Windows 包必须在 Windows x64 主机上原生构建。
