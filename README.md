# EasyInput AI HostAction v1 示例项目

## 项目简介

`easy-input-maker-ai-host-action-v1` 是基于 **EasyInput** 键盘固件的扩展，实现了 **AI HostAction v1** 协议。用户在按下编码器（Encoder）时，固件会通过 BLE/HID 向上位机发送 `ai_host_action:mic_start` 指令，触发电脑麦克风进行语音识别（示例使用 OpenAI Whisper）。

- **核心功能**
  - 将 `ai_host_action:` 前缀 + 36 位 UUID 编码为报文发送。
  - 在 `keymap.cpp` 中将 `EncoderPress` 映射为 `ActionKind::AiHostAction`。
  - 提供 BLE/HID 监听脚本 `tools/mic_ai_listener.py`，自动捕获 `ai_host_action:mic_start` 并调用本地麦克风录音。
  - 提供 Tkinter GUI `tools/ai_mic_ui.py`，可配置参数、启动/停止监听并展示 Whisper 转写结果。
  - 完整的环境初始化脚本 `tools/setup_env.sh`，一键创建虚拟环境并安装依赖。

## 目录结构

```
easy-input-maker-ai-host-action-v1/
│   CMakeLists.txt                # 项目构建入口
│   README.md                     # 本文档
│   idf.py                        # ESP-IDF 入口（用于 esp-idf-cy skill）
│
├─ components/
│   └─ keyboard/
│       ├─ include/keyboard/    # 头文件，包含 ai_host_action_protocol.h 等
│       └─ src/                 # 实现文件，keymap.cpp、config_payload.cpp 等
│
├─ host_test/                     # 单元测试文件
│
├─ main/                          # app_main.cpp（示例主入口）
│
└─ tools/                         # 辅助脚本
    │   ai_mic_ui.py               # Tkinter UI 前端
    │   mic_ai_listener.py          # BLE 监听并调用 Whisper
    │   ai_host_action_integration.py # 简化调用的整合示例
    │   setup_env.sh                # 环境初始化脚本
    └─ config.ini (generated)       # 运行时配置文件（由 setup_env.sh 创建）
```

## 前置条件

- **ESP‑IDF** 已安装（推荐 5.5.5），路径已在环境变量 `IDF_PATH` 中。
- **Python 3.10+**（本项目使用管理版 3.13.12），建议使用项目自带的虚拟环境。
- **BLE/HID 驱动**：Windows 需要安装 Silicon Labs CP210x 驱动，Linux/macOS 需要 `bluez`。
- **OpenAI API Key**（用于 Whisper）或自行替换为其他语音识别服务的 SDK。

## 环境搭建（一步完成）

```bash
# 进入项目根目录
cd D:/Easyinput/easy-input-maker-ai-host-action-v1

# 运行初始化脚本（会创建 .venv 并安装依赖）
bash tools/setup_env.sh
```

脚本会生成 `tools/config.ini`，请编辑其中的占位符：
- `<YOUR_BLE_DEVICE_NAME>`：EasyInput 设备的 BLE 名称（默认为 `EasyInput`）。
- `<YOUR_OPENAI_API_KEY>`：OpenAI API 密钥。
- 如需更改采样率、录音时长等参数，也可直接在 `config.ini` 中修改。

## 编译 & 烧录

```bash
# 编译固件
esp-idf-cy build -p D:/Easyinput/easy-input-maker-ai-host-action-v1

# 烧录至 COM3（请先确认已按下 BOOT 键进入下载模式）
esp-idf-cy flash -p D:/Easyinput/easy-input-maker-ai-host-action-v1 -P COM3
```

> **注意**：烧录前请确保板子已正确进入下载模式，且 `COM3` 为实际检测到的串口号。

## 运行 & 监控

```bash
# 实时监控串口日志（可选）
idf.py -C D:/Easyinput/easy-input-maker-ai-host-action-v1 monitor -p COM3 -b 115200 -t 30
```

若日志中出现 `AI HostAction received: ai_host_action:mic_start`，说明编码器按键已成功触发。

## 使用 UI 进行语音识别

```bash
# 启动图形界面（需要先激活虚拟环境）
source .venv/Scripts/activate   # Windows Git Bash
# 或 source .venv/bin/activate  # Linux/macOS
python tools/ai_mic_ui.py
```

- 点击 **“开始监听”**，程序会扫描 BLE 设备并订阅特征。
- 当按下编码器后，UI 会自动开始录音，完成后将调用 Whisper 并在 **“识别结果”** 区域显示文字。

## 替换为其他语音识别服务

如果不想使用 OpenAI Whisper，可自行修改 `tools/mic_ai_listener.py` 中的 `transcribe_audio` 函数，实现对接科大讯飞、Azure Speech、Google Speech‑to‑Text 等 SDK，只需保证返回的文本字符串即可。

## 常见问题

| 问题 | 解决方案 |
|------|----------|
| **无法检测到 BLE 设备** | 确认电脑已打开蓝牙，EasyInput 固件的 BLE 服务已启用；在 Windows 检查设备管理器中的 Bluetooth Radio；在 Linux 执行 `sudo btmgmt find` 确认设备可见。 |
| **`ai_host_action:mic_start` 未上报** | 检查 `keymap.cpp` 中 `EncoderPress` 是否映射为 `ActionKind::AiHostAction`，并确保 `CONFIG_AI_HOST_ACTION_V1=y` 已写入 `sdkconfig.defaults`，重新编译烧录。 |
| **Whisper 转写报错** | 确认 `OPENAI_API_KEY` 已在 `config.ini` 中配置且网络可以访问 `api.openai.com`；若使用代理，请在环境变量 `HTTPS_PROXY` 中设置。 |
| **串口监控卡死** | 使用 `idf.py monitor -p COM3 -b 115200 -t 0`（不限制时间）手动观察；确保驱动已正确安装，尝试更换 USB 线或使用不同的 COM 口。 |

## 贡献指南

1. Fork 本仓库。
2. 在 `components/keyboard/` 目录下实现新的 `ActionKind`（例如 `AiHostActionV2`），并在 `keymap.cpp` 中映射相应的按键。
3. 添加对应的单元测试文件至 `host_test/`。
4. 提交 Pull Request，CI 将自动执行 `idf.py test` 检查。

---

**项目维护者**：小李子（GitHub:https://github.com/jack-pq）
**联系方式**：请通过工作空间内的 Issue 联系。
