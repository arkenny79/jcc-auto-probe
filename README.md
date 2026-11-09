# 多游戏自动拨测：金铲铲之战与王者荣耀

本仓库将两款游戏作为同一个自动拨测项目下的独立执行器，共用设备绑定、采集、生命周期事件、单轮/多轮编排和报告。仓库名暂时保留 `jcc-auto-probe`，原有金铲铲入口继续使用。

**当前版本：2026-11-09 整合版。代码接入不等于实机验收。** 本次完成离线测试和启动命令检查，尚未在整合后的仓库重新跑实机全流程。首次部署需要另外准备配置、外部工具和王者模型。各功能的实现情况、历史证据和未验证项见 [实现与验证状态](docs/IMPLEMENTATION_STATUS.md)。

## 工程结构

```text
minipilot-main/
  integration/
    game_profiles.py             两款游戏的包名、入口和前置目标
    game_probe_orchestrator.py    单轮编排
    multi_round_orchestrator.py   同一游戏的多轮编排与重试
    probe_report.py               按游戏生成报告
  mini_pilot/                    前置视觉 Agent、设备与采集接口
  unicapture/                    录屏、抓包、QoE 和后处理
game-agent/
  jinchanchan/                    金铲铲执行器及启动脚本
  wzry/                          王者执行器、OCR 前置导航及启动脚本
  scheduler/                     历史跨游戏调度实验，默认 DRY_RUN
docs/                            架构、验证状态、来源与历史记录
```

共用流程：启动采集 → 启动游戏/前置导航 → `MATCH_READY` → 对局执行 → `MATCH_FINISHED` → 停止采集 → 后处理 → `manifest.json` 和 `report.md`。

金铲铲使用视觉 Agent 处理前置页面；王者默认使用 ADB、OCR 和开始识别模型进入单人练习场。两款游戏分别维护局内策略和结束判断。共用 JSONL 生命周期接口，不合并模型或 Python 运行环境。

## 部署

以下命令从仓库根目录运行。当前适配目标是 Windows、PowerShell、雷电模拟器。王者已知布局为横屏 `1920x1080`；其他布局未验收。设备序列号以本机 `adb devices` 为准。

1. 安装采集环境和本地配置：

```powershell
py -3.12 -m venv .\minipilot-main\.venv
& .\minipilot-main\.venv\Scripts\python.exe -m pip install -r .\minipilot-main\requirements.txt
Copy-Item .\minipilot-main\mini_pilot.config.example.json .\minipilot-main\mini_pilot.config.json
# 修改自己的配置，已有配置时不要重复覆盖。
```

2. 安装金铲铲环境：

```powershell
py -3.12 -m venv .\game-agent\.venv
& .\game-agent\.venv\Scripts\python.exe -m pip install -r .\game-agent\jinchanchan\requirements.txt
```

3. 王者使用独立 Python 3.10 环境。依赖、模型清单和校验方法见 [王者部署说明](game-agent/wzry/README.md)。仅运行金铲铲时不需要安装王者依赖。

4. 配置 ADB，准备 scrcpy、FFmpeg 和设备侧采集工具。无 Root 抓包需要项目配套的定制 PCAPdroid；有 Root 的设备可按采集工程说明使用 tcpdump。具体配置见 [采集工程说明](minipilot-main/README.md)。这些工具不通过 pip 自动安装，也不包含在本次上传中。

```powershell
$env:MINI_PILOT_ADB_BIN = 'D:\leidian\LDPlayer14\adb.exe' # 按本机路径修改
& $env:MINI_PILOT_ADB_BIN devices
```

两个启动脚本默认使用本仓库的 `minipilot-main`；`MINI_PILOT_ROOT` 可覆盖采集工程目录。检查旧环境变量，避免意外运行到旧部署。`-GamePython` 可覆盖对应游戏的解释器。

## 运行入口

先预览命令。`-DryRun` 不启动游戏、不连接设备、不执行采集，也不代表依赖检查通过：

```powershell
.\game-agent\jinchanchan\run_capture_probe.ps1 -DryRun -RunId jcc-preview
.\game-agent\wzry\run_capture_probe.ps1 -DryRun -RunId wzry-preview
```

准备好环境后运行一轮，示例设备号需按本机修改：

```powershell
.\game-agent\jinchanchan\run_capture_probe.ps1 -DeviceId '127.0.0.1:5555' -RunId jcc-round-01
.\game-agent\wzry\run_capture_probe.ps1 -DeviceId 'emulator-5554' -RunId wzry-round-01
```

同一模拟器上依次运行，等待上一条结束。金铲铲默认目标为标准排位；王者默认目标为单人练习场。真实登录、验证码、实名和账号安全页面需人工处理；已登录状态可以尝试从“开始游戏”进入大厅。

同一游戏的多轮入口已有实现，**王者自动连续两轮尚待本版本实机验收**：

```powershell
.\game-agent\wzry\run_capture_probe.ps1 -Rounds 2 -SessionId wzry-two-rounds
.\game-agent\jinchanchan\run_capture_probe.ps1 -Rounds 2 -SessionId jcc-two-rounds
```

多轮编排在上一轮结束后使用结算返回目标，失败时切换恢复目标；重试耗尽就停止。它不等于无人值守可靠性已验证，也不提供两款游戏自动交替调度。使用不同的运行/会话标识保存每次实验。

## 结果与判断

默认产物位于 `minipilot-main/runs/<run_id>-<game_id>-external/`：

- `manifest.json`：运行、采集、后处理状态及结束来源。
- `events/game_driver.jsonl`：权威生命周期事件；不要用普通输出日志替代。
- `capture/`：视频、抓包、逐秒 QoE、流级结果和后处理摘要。
- `report.md`：按游戏生成的中文汇总，标记缺失产物与后处理问题。

多轮会话另有 `<session_id>-multi-round/session_manifest.json`。`status=ok` 表示流程正常返回，仍需检查采集文件、后处理状态和告警；`result=unknown` 不代表胜负已识别。指标文件的存在也不等于质量准确性验收。

## 文档与来源

- [实现与验证状态](docs/IMPLEMENTATION_STATUS.md)：本次验证记录、未验证/未实现列表及下一步验收。
- [王者采集接口](game-agent/wzry/CAPTURE_INTEGRATION.md)：事件、控制权和结果语义。
- [来源说明](docs/SOURCE_NOTES.md)：迁入版本、上游署名及模型发布状态。
- [原金铲铲文档](docs/JCC_LEGACY.md)、[架构与技术路线](docs/架构与技术路线.md)、[历史工作日志](docs/组会汇报-项目工作日志.md)：保留研究过程；部署与当前完成度以本页和状态表为准。

源码和必要的小型识别素材入库；密钥配置、虚拟环境、模型、录屏、抓包、日志和运行数据不入库。
