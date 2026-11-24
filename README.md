# 10RemoteHub

把电视遥控器变成 **macOS / Windows** 上的按键面板、九键输入与语音入口：配套 `USB 0416:0602`、系统识别为 HID 键盘／媒体键的 USB 与蓝牙遥控器，以及经外接 USB 蓝牙棒直连的 BLE 语音遥控器。

- **协议优先，一条接入路径**：在「我的遥控器 → 添加遥控器」试按一次，App 用大白话告诉你这只遥控器属于哪一档（✅ 直接接入 / 🔧 点一下就能接入 / ⚠️ 目前无法自动接入 / ❓ 看不到设备）。随后导入你手里这只遥控器的正面照片，逐个点按键位置并按实体键学习。不限外观和键数，同一只遥控器可同时绑定 USB 接收器和蓝牙。
- **遥控器麦克风**：内置 `USB 0416:0602`、BLE `FIBER0D`（G.722.1）、BLE `ATV Voice`（IMA-ADPCM）三种语音协议。在照片上标出语音键后，遥控器的声音在本机解码，送入虚拟麦克风交给你选的识别软件；其他遥控器的语音键触发识别软件，使用电脑麦克风。
- **动作**：每个普通键可设单击、双击、长按，支持快捷键、整键跟手鼠标键、打开/收起应用、打开文件、输入文本（可选自动回车）、多步动作链、3×3 九宫格、键位速查、九键拼音输入、按前台应用自动切换按键层，以及界面控件点击、应用菜单命令与聊天输入框自动聚焦。
- **隐私**：按键、配置和音频只在本机处理；诊断日志不记录按键历史、文本、录音、序列号或个人路径。

当前版本 `0.3.1`，依赖基线 Python 3.12 / PySide6 6.12.0。正式发布支持 macOS 14+（Apple 芯片）与 Windows 10/11 x64；Linux 支持已在源码主干并通过离屏测试，尚未真机验证，暂不提供 Linux 安装包。

本仓库是最新版本的独立源码快照，不含开发仓库的旧历史。安装前请阅读对应平台说明，并核对发布页的 `SHA256SUMS`。

## 界面预览

按键映射：选择遥控器上的按键，再设置单击、双击或长按动作。

![按键映射界面，显示未分配的示例配置](desktop/docs/screenshots/mapping.png)

语音设置：选择声音来源、遥控器操作方式和识别软件快捷键。

![macOS 语音设置界面](desktop/docs/screenshots/voice.png)

截图为 macOS 界面与示例配置，不代表设备已连接。

## 安装使用

从 [GitHub Releases](https://github.com/yizhixiaoheigou/10RemoteHub/releases) 下载对应平台的发布包，并核对同页的 `SHA256SUMS`。

- **macOS**（[安装、授权、语音、更新与卸载](desktop/INSTALL-macOS.md)）：主程序拖进「应用程序」，授予「辅助功能」；通用 USB / 蓝牙遥控器再授予「输入监控」。使用配套 `0416:0602` 时安装「读取组件」，要用遥控器麦克风再装「虚拟麦克风」。添加遥控器后，App 会列出还差哪几步。
- **Windows**（[解压运行、接管接收器、蓝牙棒直连、语音与卸载](desktop/INSTALL-Windows.md)）：解压即用，通用 HID 遥控器免装驱动。
  - 系统读不全的 USB 接收器（如 `0416:0602` 的 `#` 键与遥控器麦克风），在添加窗口选中它点「让 App 接管这个接收器」，App 换上 Windows 自带的 WinUSB 驱动，不需要 Zadig，可随时一键恢复。
  - Windows 自带蓝牙读不了的 BLE 遥控器，插一只外接 USB 蓝牙棒，在添加窗口一键切换为直连模式。
  - 使用遥控器麦克风时需另装 VB-CABLE。

添加遥控器后若卡住，见 [常见问题](desktop/TROUBLESHOOTING.md) 与 [遥控器适配说明](desktop/docs/REMOTE-ADAPTATION.md)。

## 从源码验证

macOS（Apple 芯片；Intel Mac 换成 `requirements-macos-x86_64.lock`）：

```sh
cd desktop
python3.12 -m venv .venv
.venv/bin/python -m pip install --upgrade 'pip>=26.2'
.venv/bin/python -m pip install --no-deps --require-hashes -r requirements-macos-arm64.lock
.venv/bin/python -m pip install --no-deps --no-build-isolation -e .
.venv/bin/python scripts/verify.py --output ../artifacts/verification-macos.json
```

Windows PowerShell：

```powershell
cd desktop
py -3.12 -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade "pip>=26.2"
.\.venv\Scripts\python.exe -m pip install --no-deps --require-hashes -r requirements-windows-x64.lock
.\.venv\Scripts\python.exe -m pip install --no-deps --no-build-isolation -e .
.\.venv\Scripts\python.exe scripts\verify.py --output ..\artifacts\verification-windows.json
```

lock 文件已列出全部依赖及哈希，所以用 `--no-deps` 安装：Bumble 精确要求 `libusb-package==1.0.26.4`，而 lock 保留 `1.0.30.0`。Linux 开发环境使用 `requirements-linux-x86_64.lock` 运行同一套 `scripts/verify.py`，见 [INSTALL-Linux.md](desktop/INSTALL-Linux.md)。

离屏测试使用模拟设备和临时配置，不安装组件、不接管物理 USB，也不改系统音频设置。签名身份、许可材料与构建流程见 [RELEASE.md](desktop/RELEASE.md)。

## 实机验证范围

- **已实机验证**：`0416:0602` 的按键、遥控器麦克风与九键拼音输入（macOS arm64、Windows 11 x64；含 macOS `0.3.0` 照片学键、「完成设置」清单、安装虚拟麦一键重开 App，以及 Windows `0.3.1` 软件内一键接管接收器）；macOS 上蓝牙飞鼠 `620A:0407` 的照片学键、原按键屏蔽与多遥控器切换（`0.3.0`）；Windows 上经外接蓝牙棒 `0BDA:8771` 直连 `0416:0300` / `FIBER0D` 的照片学键、按键与实时语音（`0.3.0`）。
- **待实机验证**：Windows 下非 `0416:0602` 普通未知 2.4G 接收器的一键接管与 15 秒自动恢复、Windows 通用共享 HID 原按键屏蔽；macOS 读取组件空闲同步（长时闲置专项）；macOS 与 Linux 上的蓝牙棒直连；BLE `ATV Voice` 真机；Intel Mac 与 Linux 整体。离屏自动化结果不等于真机验收。
- 不承诺「任意遥控器即插即用」：只暴露厂商私有协议的设备会停在「⚠️ 目前无法自动接入」，并提供「复制给 AI 的求助」以便二次适配。

## 使用与开发资料

- [版本更新日志](CHANGELOG.md)
- [遥控器适配与边界说明](desktop/docs/REMOTE-ADAPTATION.md) · [接入流程图](desktop/docs/REMOTE-ADAPTATION-FLOW.html) · [接入记录表](desktop/docs/REMOTE-RECORD-TEMPLATE.md)
- [macOS 安装与权限](desktop/INSTALL-macOS.md) · [Windows 安装与第三方组件](desktop/INSTALL-Windows.md) · [Windows 架构与验收说明](desktop/docs/WINDOWS-TESTING.md) · [Linux 源码适配说明（待真机验证）](desktop/INSTALL-Linux.md)
- [常见问题](desktop/TROUBLESHOOTING.md) · [`0416:0602` 内置按键表](desktop/KEYMAP.md) · [架构](desktop/ARCHITECTURE.md) · [平台动作矩阵](desktop/docs/PLATFORM-ACTIONS.md)
- AI 助手操作说明：[SKILL.md](SKILL.md)（含命令行工具 `10remotehub-cli`）
- [参与开发](CONTRIBUTING.md) · [安全报告](SECURITY.md)

macOS 的读取组件与可选虚拟麦克风由用户通过标准安装器安装。Windows 的 VB-CABLE 由用户向原提供者获取，本项目不附带。

## 许可

项目原创代码采用 [MIT](LICENSE)。内置 Python 音频解码器沿用 LGPL-2.1-or-later，Apple 示例、Bumble 及其他第三方代码保留各自许可，MIT 不覆盖这些组件。详见 [第三方声明](desktop/THIRD_PARTY_NOTICES.txt)、[内置解码器来源](desktop/remotehub/codec/NOTICE.md) 和 [对应源码 / 修改版说明](desktop/licenses/SOURCE.md)。

当前遥控器图为程序生成的示意图；源码快照不含旧照片、个人诊断日志、配置或闭源解码器实验。`SOURCE-INVENTORY.json` 记录快照中各文件的 SHA-256。
