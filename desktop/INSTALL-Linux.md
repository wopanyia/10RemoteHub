# Linux 源码适配与测试指南（待真机验证）

> **发布状态说明**：Linux 端代码与打包工具链已合入主干（`0.2.0` 起，`0.3.0` 继续保留）并通过离屏自动化测试，但因当前尚未完成实体接收器真机验收，**本次正式发布暂不提供预编译 Linux 安装包**。开发者可从源码环境或自行运行 `desktop/scripts/build_linux.sh` 构建测试。

> **外接蓝牙棒直连（`0.3.0`）**：依赖 Bumble 已随源码 lock 安装，但 Linux 上尚未实机验证；附带的 `udev` 规则只覆盖 `0416:0602`，没有为外接 USB 蓝牙棒授权，也不会让系统蓝牙服务（BlueZ）放开该蓝牙棒。需要自行为蓝牙棒加 `uaccess` 规则并避免 BlueZ 占用后才可能使用。

本指南适用于 Ubuntu 22.04 LTS / 24.04 LTS、Debian 12+、Fedora 39+、Arch Linux 等主流 `x86_64`（`glibc >= 2.34`）桌面发行版，同时支持 **X11** 与 **Wayland** 会话。

> **安全原则**：`10RemoteHub` 全程以**普通桌面登录用户**身份运行，切勿使用 `sudo ./10RemoteHub` 启动图形界面（以 `root` 启动会导致无法连接当前用户的 PipeWire / PulseAudio 音频服务与桌面剪贴板）。

---

## 1. 解压与启动（本地构建包）

运行 `bash desktop/scripts/build_linux.sh` 生成 `10RemoteHub-0.3.1-linux-x86_64.tar.gz` 后，解压到任意目录（如 `~/Applications/` 或 `/opt/`）：

```bash
mkdir -p ~/Applications
tar -xzf 10RemoteHub-0.3.1-linux-x86_64.tar.gz -C ~/Applications
~/Applications/10RemoteHub/10RemoteHub
```

系统基础图形与音频运行库（桌面发行版通常已预装；极简环境可按需补齐）：

- **Ubuntu / Debian**：
  ```bash
  sudo apt update
  sudo apt install -y libxcb-cursor0 libxkbcommon-x11-0 libxcb-icccm4 libxcb-keysyms1 \
      libxcb-shape0 libxcb-xinerama0 libpulse0 pulseaudio-utils libglib2.0-bin
  ```
- **Fedora**：
  ```bash
  sudo dnf install -y xcb-util-cursor libxkbcommon-x11 pulseaudio-utils glib2
  ```

---

## 2. 配置一次性 `udev` 设备权限规则（推荐，仅需执行一次）

适配遥控器 USB 接收器（`0416:0602`）包含标准 HID 按键接口、扩展键盘接口（含 `#` 号键）和私有语音音频接口。安装附带的最小权限 `udev` 规则（使用 `TAG+="uaccess"`，仅向当前本地登录会话授权，不开放全局 `0666`）：

```bash
sudo install -Dm644 ~/Applications/10RemoteHub/udev/70-10remotehub.rules \
    /etc/udev/rules.d/70-10remotehub.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

执行完成后，**拔出并重新插入一次 USB 接收器**（Wayland 用户若首次启用 `/dev/uinput`，建议注销并重新登录桌面会话一次），回到应用「通用设置」点击 **「重新检查权限与连接」** 即可看到三个绿色的 **「已安装 / 已授权」** 状态标。

> 您也可以直接在应用内打开 **「通用设置 → 设备连接」**，点击 **「复制 udev 规则安装命令」** 粘贴到终端运行。

---

## 3. 配置遥控器语音输入（零驱动、免重启）

Linux 无需安装第三方内核声卡驱动，也无需重启系统：

1. 打开 **「语音设置」**，将「语音来源」设为 **「遥控器麦克风」**。
2. 在「4. 遥控器音频连接」卡片中点击 **「启用虚拟麦克风」**：
   - 程序会自动通过 `pactl`（兼容 PipeWire-Pulse 与原生 PulseAudio）或 `pw-loopback` 在用户态创建 `10RemoteHub Audio Feed` 接收端与 **`10RemoteHub Microphone`** 虚拟麦克风输入源；退出程序时自动清理。
3. 将「连接方式」保持为 **「录音时切换系统默认麦克风」**：
   - 按住或按下遥控器语音键时，程序会自动把系统默认输入源临时切换至 `10RemoteHub Microphone`，松开结束时自动恢复原麦克风（若您在录音中途手动更改了系统默认麦克风，程序会尊重您的手动选择不强行覆盖）。
4. 在您的外部语音输入法 / 听写工具（如豆包桌面端、Fcitx5 语音插件、Whisper 桌面工具等）中：
   - 将麦克风设为 **「系统默认」**（或固定选 `10RemoteHub Microphone`）；
   - 在 `10RemoteHub`「识别软件触发」中录入该软件的听写唤起快捷键。

---

## 4. X11 与 Wayland 会话能力说明

| 功能项 | X11 会话（GNOME on Xorg / KDE X11 / XFCE） | Wayland 会话（Ubuntu Wayland / KDE Wayland） |
| --- | --- | --- |
| 遥控器全键位独占采集（含 `#` 号键）与语音流 | 完整支持（`libusb` 接管并退出时自动归还内核 `usbhid`） | 完整支持 |
| 全局快捷键、多媒体键、滚轮与鼠标按键模拟 | 完整支持（`XTest`） | 完整支持（需配置上方 `udev` 规则启用 `/dev/uinput`） |
| 输入文本动作（剪贴板粘贴） | 完整支持（识别到终端窗口自动改用 `Ctrl+Shift+V`） | 完整支持（安装 `wl-clipboard` 时同步写入 `wl-copy`） |
| 按前台应用自动切换按键层（App Layers） | 完整支持（EWMH `_NET_ACTIVE_WINDOW` + `.desktop` 匹配） | 受 Wayland 跨应用隔离限制，固定使用全局默认层 |
| 打开 / 收起 App、最小化与关闭当前窗口 | 完整支持（EWMH 窗口管理消息） | 「打开 / 收起 App」仅执行启动，不支持跨应用收起/关闭 |
| 九宫格面板、键位速查浮层、方案切换顶部提示 | 完整支持（不抢焦点、点击穿透） | 完整支持 |
| 登录后自动启动 | `~/.config/autostart/org.remotehub.desktop.desktop` | `~/.config/autostart/org.remotehub.desktop.desktop` |

---

## 5. 卸载与清理

Linux 版不向系统目录写入任何后台守护进程：

1. 如开启过「登录后自动启动」，可在「通用设置」取消勾选，或删除 `~/.config/autostart/org.remotehub.desktop.desktop`。
2. 如安装过 `udev` 规则，可执行：
   ```bash
   sudo rm -f /etc/udev/rules.d/70-10remotehub.rules
   sudo udevadm control --reload-rules
   ```
3. 删除解压的应用目录（如 `~/Applications/10RemoteHub`）；个人按键方案与日志保存在 `~/.config/RemoteHub`，按需保留或删除即可。
