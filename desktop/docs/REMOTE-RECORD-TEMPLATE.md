---
knowledge-role: template
knowledge-status: current
---

# 遥控器接入记录表

协助用户接入新遥控器的 AI 边做边填这张表，做完把整张表发回项目维护者。每个阶段只填亲眼看到的结果，没做的写 `not_run`，不要推测。各阶段的含义见 [REMOTE-ADAPTATION.md](REMOTE-ADAPTATION.md) 第 3 节。

**不要填写**：设备序列号、个人目录路径、按键记录、音频内容、聊天文字、完整配置文件。可以附「复制接入诊断」生成的摘要，它已经脱敏。

```yaml
schema: remote-record/1
date: 2026-11-20
app: {kind: release|source, version: "0.3.1", entry: "main-window (python -m remotehub)"}
platform: {os: "macOS 15.1 / Windows 11 24H2", arch: arm64|x64}

device:
  transport: USB|蓝牙
  vid_pid: "XXXX:XXXX"          # App 列表或系统设备信息里显示的厂商/产品编号
  name_hint: ""                 # 系统里显示的名字、外壳上的型号
  receiver: ""                  # USB 接收器 / 电脑内置蓝牙 / 外接蓝牙适配器型号
  has_microphone: yes|no|unknown

stages:
  s0_system_connection:
    result: pass|fail|not_run
    stable: yes|no              # 连接是否反复断开
    note: ""
  s1_app_visible:
    result: pass|fail|not_run
    interfaces_opened: "n/m"    # 试按时状态栏显示的「已打开 n/m 个按键接口」
    note: ""
  s2_key_trial:
    result: pass|fail|not_run
    keys_tested: 0
    down_up_pairs: 0            # 收到完整按下+松开的键数
    down_only: []               # 只收到按下的键
    duplicates: []              # 不同实体键却得到同一键码
    no_signal: []               # 按了没有任何信号的键
    main_keyboard_isolated: yes|no|not_run   # 主键盘按同名键时，所选设备没有收到
    original_key_fires: yes|no              # 原键行为（如音量、翻页）是否同时生效
  s3_preset_or_learning:
    result: preset|manual|fail|not_run
    keys_learned: 0
    keys_total_on_remote: 0
  s4_save_reconnect:
    result: pass|fail|not_run
    stable_identity: yes|no     # 诊断里的 stable_identity_available
    auto_reconnect: yes|no|not_run
  s5_action_effect:
    result: pass|fail|not_run
    tried: [{key: "", action: "", observed: ""}]
  s6_voice_trigger:
    result: pass|fail|skipped|not_run
    recognizer: ""              # 例如 微信输入法
    shortcut: ""                # 例如 Alt，按住式/开关式
    text_appeared: yes|no       # 由用户确认
  s7_remote_microphone:
    result: system_input|private|not_present|not_run
    system_exposes_input: yes|no|unknown   # 识别软件能否直接选到遥控器麦克风
    private_protocol:           # 仅在明确授权采集后填写
      control_found: yes|no|not_run
      packets_seen: yes|no|not_run
      reassembled: yes|no|not_run
      decoded_pcm: yes|no|not_run
      routed_to_recognizer: yes|no|not_run
      reconnect_ok: yes|no|not_run

blocked_at: ""                  # 第一处卡住的阶段，例如 s1_app_visible
boundary_case: ""               # App 判断的边界：private_usb / vendor_page / gatt_only / system_control / gamepad_buttons，见 SKILL.md 6.5
diagnostics: |
  # 粘贴「复制接入诊断」的内容
proposed_next: preset|adapter|limitation|none
# preset:     标准 HID 已能稳定读键，补正面图、键位和键表做成预设
# adapter:    需要新读取端或私有音频适配器（附最小样本说明，不附原始音频）
# limitation: 设备没有可区分的信号或系统不暴露接口，记为限制
```

## 维护者收到后

| `proposed_next` | 处理 |
|---|---|
| `preset` | 在 `desktop/remotehub/models/` 新增型号文件（身份、正面图、键位、键表、试按键），补 `tests/test_device_models.py`，到实机复核试按 |
| `adapter` | 按 REMOTE-ADAPTATION.md 的读取端约定实现，并补样本回放测试。实物按键、App 动作、断开重连分别验收 |
| `limitation` | 写入设备限制说明，不改代码 |
