# 游戏执行器

金铲铲和王者同属一个拨测项目，通过独立子进程接入 `../minipilot-main/integration`。两个执行器共用事件协议和采集总控，各自维护依赖、前置导航和游戏策略。

| 目录 | 当前入口 | 运行环境 |
| --- | --- | --- |
| `jinchanchan` | `run_capture_probe.ps1` → `auto_loop.py --mode external` | Python 3.12，默认 `game-agent/.venv` |
| `wzry` | `run_capture_probe.ps1` → `train.py --mode external` | Python 3.10，默认 `game-agent/wzry/.venv` |
| `scheduler` | 历史画面识别与跨游戏调度实验 | 默认 `DRY_RUN=True`，不作为正式拨测入口 |

部署和运行命令见 [仓库说明](../README.md) 与 [王者说明](wzry/README.md)。当前双游戏整合已完成离线验证，实机与未实现事项见 [状态表](../docs/IMPLEMENTATION_STATUS.md)。

`scheduler` 不承担当前采集编排，也未验证自动交替运行两款游戏。无需修改它即可使用两个独立的拨测入口。
