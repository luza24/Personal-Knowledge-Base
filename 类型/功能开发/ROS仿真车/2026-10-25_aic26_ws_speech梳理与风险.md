---
title: aic26_ws_speech 全工作区梳理与语音链路风险（分析中止）
date: 2026-10-25
updated: 2026-10-25
project: ROS仿真车
type: 功能开发
themes: [语音与交互]
tags: [ros, 真车, deepseek]
status: 进行中
---

# aic26_ws_speech 全工作区梳理与语音链路风险（分析中止）

> 整理日期：2026年10月25日 12:26
> 项目：ROS仿真车 ｜ 状态：进行中 ｜ 标签：`功能开发` `ros` `真车` `deepseek`

## 背景 / 目标

先解决选型问题：分析实车语音工作区 `aic26_ws_speech`，用 `deepseek-flash` 还是 `deepseek-v4-pro`。
确定采用混合方案（子 agent 用 flash 扫描、主模型用 pro 深审）后，把范围定为「全工作区梳理 + 风险清单」，
结果输出为桌面 Markdown 报告。

3 个 flash 子 agent 已完成只读扫描并返回结果；用户随后叫停，深审与报告落盘未执行。

## 环境信息

- 车端：Ubuntu 20.04 + ROS Noetic，Intel N100（4 核），`ssh mowen@192.168.1.15`（密钥登录）
- 工作区：`/home/mowen/aic26_ws_speech`，分支 `feature/speech-announcement`，HEAD `97f43a0`
- 本机：Windows 11；PC 侧播报配套代码 `C:\Users\luza\Desktop\aic26_speech_pc`
- 模型接入：Claude Code 走 DeepSeek Anthropic 兼容端点
  - `deepseek-flash` = V4.1-Flash（1M 上下文、支持图像输入）
  - `deepseek-v4-pro` = V4-Pro（**无图像输入**）
  - 配置在 `settings.json` 与 Windows 用户环境变量两处，改后需完全重开终端；子 agent 固定 `CLAUDE_CODE_SUBAGENT_MODEL=deepseek-flash`
- 扫描时的未提交改动：18 个文件、+164/−70，无暂存、无未跟踪文件。
  ⚠️ 之后另有「导航回退 A」会话把导航包恢复到 `7cf1476`，本记录的改动面与部分导航风险**需按当前工作区复核**。

## 报错记录

- 本次分析无程序报错；因用户主动中止而未产出报告。
- 选型约束：pro 不能读图；主模型从 flash 切到 pro 需要改两处配置并重开会话，不能在同一次会话里无缝切换。

## 涉及文件

- 车端工作区：`/home/mowen/aic26_ws_speech/src`（10 个 `aic26_*` 包，约 6.7k 行 py/cpp）
- 语音相关：`src/aic26_decision/scripts/mission_common.py`、`mission_controller.py`、`demo_cycle.py`；
  `src/aic26_utils/scripts/{speech_broadcaster.py,kokoro_worker.py}`、`launch/speech.launch`；
  `src/aic26_utils/pc_bridge/`（`speech_relay.py` 等）
- PC 侧：`C:\Users\luza\Desktop\aic26_speech_pc\{speech_bridge.py,speech_relay.py,run_bridge.cmd,README.md}`

## 分析方案与扫描结果

### 1. 模型选型结论

| 方案 | 适用 | 代价 / 限制 |
|---|---|---|
| 全程 flash | 结构梳理、定位调用链、常规摘要 | 跨进程/跨机器时序与失败路径的深挖可能不足 |
| 全程 pro | 架构判断、风险审查、写报告 | 更慢更贵；无图像输入，看图要切回 flash |
| **混合（选定）** | 子 agent flash 扫描 + 主模型 pro 深审 | 切换需重开会话；探索阶段速度损失小 |

判断依据：代码规模（6.7k 行）对 1M 上下文不是瓶颈，差异只在单步推理深度；难点是播报、决策状态机、
PC relay 三者的时序与兜底，属于 pro 的主场。

### 2. 执行方式

3 个 Explore 子 agent 并行、全程只读 SSH：
① 包结构与 launch/节点图；② 端到端语音链路与脆弱点；③ 未提交改动与全工作区风险模式。
未修改车上或本机任何文件，未写报告。

### 3. 包结构与启动链

唯一总入口 `src/aic26_bringup/launch/bringup.launch`（`mode` + `backend` 两个参数）。

| 包 | 职责 / 主要节点 |
|---|---|
| `aic26_bringup` | 顶层 launch 编排，无节点 |
| `aic26_cobot` | 真机驱动：chassis、imu、lidar、camera、hardware_status |
| `aic26_decision` | mission_controller、waypoint_follower、demo_cycle、capture_cycle；公共库 `mission_common` |
| `aic26_navigation` | map_server、amcl、move_base、rviz |
| `aic26_perception` | traffic_light / people / car 三个 ONNX 检测器 + `perception_common` |
| `aic26_slam` | gmapping、保存地图 |
| `aic26_description` | cobot 与 Gazebo armbot 的 xacro 模型 |
| `aic26_gazebo` | world、models、traffic_controller、day_night、2 个 C++ visual plugin |
| `aic26_msgs` | `TrafficLightCommand`、`TrafficLightState` |
| `aic26_utils` | camera_view、speech_broadcaster、kokoro_worker、pc_bridge |

- `mode:=mission`：backend + navigation + 3 个感知 + mission，完整巡检；
  `demo`：假检测结果 + 循环播报；`capture`：沿路线截图采数据集；`waypoints`：只验证地图/定位/规划；
  `mapping`：backend + gmapping。`backend:=real` 起 cobot 驱动，`sim` 起 Gazebo。
- 节点链路：底盘/Gazebo → `/odom` `/scan` `/camera/image_raw` `/tf` → amcl/move_base/检测器；
  move_base 发 `/cmd_vel`；decision 通过 `RobotDriver` 发 MoveBaseGoal，原地转向直接发 `/cmd_vel`；
  mission 发 `/perception/detect`，检测器回 `/perception/result` 或 `/perception/traffic_light`；
  traffic_controller 发 `/traffic_light/state`；`/speech` 由 mission/demo 发布，broadcaster 调 Kokoro 播放。
- 所有 launch 无 `<remap>`，话题通过节点私有参数 `~xxx_topic` 绑定。

### 4. 端到端语音链路

```text
mission_controller(report/inspect) / demo_cycle(fake_vision)
  → mission_common.SpeechAnnouncer → /speech (std_msgs/String)
  → speech_broadcaster（queue=1）--stdin JSONL--> kokoro_worker
  → Kokoro KPipeline 24kHz → 分段临时 wav → subprocess paplay → PulseAudio/蓝牙 sink

PC 旁路：
  /speech → speech_relay(stdout "AIC26_SPEECH {json}") --SSH 标准流-->
  speech_bridge → KokoroPlayer → sounddevice 播放（Windows 默认设备）
```

- 车上与 PC 各自独立：车上走 stdin/stdout + paplay；PC 无 ROS，靠 `scp` relay 到 `/tmp` 再 `ssh -T` 收标准流。
- 协议：`AIC26_SPEECH ` 前缀 + ASCII JSON，类型 boot/ready/speech/shutdown；重连退避 2→30 s，模型常驻。
- `speech.launch` 是独立入口，`bringup` 并未引用它；车端当前实际以 PC 播报为主。

### 5. 语音链路脆弱点

| # | 问题 | 证据 |
|---|---|---|
| 1 | bridge 加载 Kokoro 时死循环等待、`proc.wait()` 永久阻塞 | `speech_bridge.py:80-84,358` |
| 2 | relay 在 ROS master 缺失时卡在 `init_node`，只有 stdin EOF 看门狗，不会重连 | `speech_relay.py:53-71,93` |
| 3 | 播报端未启动时 `/speech` 无订阅者，消息静默丢失，无 ack/回退 | `speech.launch` 未并入 bringup；`aic26_utils/README:55-56` |
| 4 | queue=1 + 就绪前单槽 pending + worker 崩溃 5 s 重启窗口内直接丢消息 | `speech_broadcaster.py:81,135-136,175-178,230-238` |
| 5 | worker 用 `communicate()` 等整段播完；合成无超时，首次模型下载可长时间挂起 | `kokoro_worker.py:179-224` |
| 6 | PC 端 `submit` 先 gen++ 再 `sd.stop`，可能截断刚开播的新句；`kill_player` 与 `play` 之间有竞态 | `speech_bridge.py:121-130,179-203,216-250` |
| 7 | 异常被吞：`_stop_audio/_stream_active`、relay `emit`；`check_setup` 只查文件存在 | `speech_bridge.py:98-101,210-224`；`speech_relay.py:33-37` |
| 8 | 播报时机紧：触发检测后只等 `detect_wait=1s` 即朗读，OCR 较慢会误报“未识别” | `mission_controller.py:173,224-230` |
| 9 | 车端与 PC bridge 可能同时响；worker 无条件加 paplay 参数，换 aplay 即失败；sink 不存在只记日志 | `speech_broadcaster.py:89-96`；`kokoro_worker.py:229-232` |

### 6. 全工作区风险清单（按严重度）

| 级别 | 问题 | 证据 | 后果 / 验证 |
|---|---|---|---|
| 高 | 后端校验不区分 `backend`：只拒绝 `/chassis` 与 `/gazebo` 同时存在，任一即可放行 | `mission_common.py:263-272`；四个调用点未传 backend（`capture_cycle.py:261`、`demo_cycle.py:169`、`mission_controller.py:246`、`waypoint_follower.py:34`） | 实车启动失败但残留 `/gazebo` 时会驱动仿真车；反之亦然。**待实车+仿真验证** |
| 高 | 仿真底盘换 skid_steer 后丢失命令超时（旧 `<cmdTimeout>0.5` 删除） | `gazebo_car_union.xacro:14-33` | `/cmd_vel` 中断后仿真车保持最后速度。**待仿真验证** |
| 高 | 红绿灯 fail-open：无数据超过 10 s 或等待超时都 “passing anyway” | `mission_common.py:443-449` | 检测器掉线会闯红灯。**待实车验证** |
| 高 | 30 s 启动门禁过紧且失败即退出（退出码 0） | 四个 decision launch（如 `mission.launch:52`）；`mission_common.py:292-301` | N100 冷启动慢时任务静默不跑，roslaunch 不报错。**待实车验证** |
| 中 | 转向精度下降：`turn_speed/turn_tolerance=0.8/0.10`，`turn_min_speed=0.15` | decision launch；`mission_common.py:375-397` | 约 5.7° 对准误差，capture/inspect 可能偏。**待实车验证** |
| 中 | 非完整约束与实车不一致：删除 `max_vel_y/acc_lim_y`，`weight_kinematics_nh=1000`；实车底盘仍支持 vy | `local_planner_params.yaml:3-18`；`chassis.yaml` | 窄路可能多转/失败，仿真不再覆盖横移。**待实车+仿真验证** |
| 中 | footprint 在 4 个 bringup launch 与 `navigation.launch:20` 各写一份，`footprint_padding=0` | 同上 | 漏改一处即碰撞或规划异常；膨胀余量 0.09/0.15 偏小。**待实车验证** |
| 中 | `aic26_decision/package.xml` 未声明新 import 的 `rosnode`、`cv_bridge`、`yaml`、`numpy` | `package.xml:12-22`；`mission_common.py:33` 等 | 干净 rosdep/最小安装可能 import 失败。**待构建验证** |
| 中 | 车端语音链路未就绪：`~/aic26_kokoro_venv` 不存在，系统 Python 3.8 无 kokoro/sounddevice | `speech.launch:20`；`speech_broadcaster.py:98-107` | `/speech` 仍发布但无声；broadcaster 自行禁用不崩溃。**待功能验证** |
| 低 | 图像回调做 `imgmsg_to_cv2+cv2.imwrite`，GUI 默认开启；`subprocess.call` 无 timeout | `mission_controller.py:327-341,357-364`；`capture_cycle.py:226`、`demo_cycle.py:161` | 30 fps 下可能阻塞；master 异常时循环卡住 |
| 低 | 配置漂移/硬编码：代码默认 0.05/60 与 launch 的 0.10/30 不一致；设备名 `/dev/*` 写死；`run_bridge.cmd` 写死 `C:\Users\luza` | 多处 | 维护时易漏改；未发现硬编码 IP、备份文件或 TODO |

**扫描时的未提交改动面**（+164/−70）：bringup 4 个 launch 加 `footprint` 透传；decision 10 文件
（启动门禁、转向闭环、导航超时、真实起点）；navigation 3 文件；description xacro 改 skid_steer。
与 HEAD 相比无节点/话题/服务增删，新增参数 `footprint`、`turn_min_speed`。

## 结果与结论

- 扫描已完成，**报告未写、未落盘**；用户叫停后未继续。车上与本机均零改动。
- 风险清单反映的是扫描时（导航回退 A 之前/独立于 A）的工作区状态；A 已把导航包恢复到 `7cf1476`，
  因此上表“转向参数、footprint、非完整约束”等条目**需按当前工作区重新核对**，不要直接照单修。
- 若恢复分析：先用 `/model` 或新会话切到 pro，再把本记录第 3~6 节作为已完成的扫描输入，
  只做深审与报告撰写，避免重复扫描；报告建议放 `C:\Users\luza\Desktop\`。

## 相关记录

- [AIC26 小车检查点语音播报与 Windows 本机转发](2026-10-23_小车检查点语音播报与本机转发.md)
  —— 播报链路的设计与实测，本记录在其基础上补全脆弱点清单。
- [aic26_ws_speech 导航回退 A 与任务点转向偏差排查](2026-10-25_导航回退A与转向偏差排查.md)
  —— 同一工作区的后续会话，导航包已回退；本记录的风险条目需与其对照复核。
- [把大型 ROS 项目交给 ChatGPT 分析：打包与推理强度选型](../开发环境/2026-10-25_给ChatGPT分析大项目的打包与强度选型.md)
  —— 同一类「用 LLM 分析大型 ROS 工程」的模型/强度选型经验，可互相参照。

## 变更记录

- 2026-10-25 12:26 首次整理
