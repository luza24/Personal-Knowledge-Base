---
title: ebike_pano 与 rc job_parking 冲突分析与可移植性评估
date: 2026-11-28
updated: 2026-11-28
project: ROS仿真车
type: 方案选型
themes: [感知与视觉, 方案与选型]
tags: [ros, perception, yolo, openvino, 真车]
status: 部分解决
---

# ebike_pano 与 rc job_parking 冲突分析与可移植性评估

> 整理日期：2026年11月28日 21:24
> 项目：ROS仿真车 ｜ 类型：方案选型 ｜ 状态：部分解决 ｜ 主题：`感知与视觉`、`方案与选型` ｜ 标签：`ros`、`perception`、`yolo`、`openvino`、`真车`

## 背景 / 目标

同一台实车上有两套并列代码：`~/aic26_ws_112`（ROS1 Noetic 巡检栈）和 `~/Desktop/rc`（另一赛项工程），两套轮流运行、共用同一台 Orbbec DaBai DCW2 与同一物理安装。

两边各有一套停车区电动车识别：

- aic26 `ebike_pano`：停车区航点原地转 3 个视角，每视角取最清晰帧，末视角拼接全景后推理 `ebike.onnx`（down/normal），输出"停车区内电动车正常 N 辆，倒伏 M 辆"。
- rc `job_parking`：640 模型 + zoom 多尺度切片 + 3 帧 burst 投票 + 单视图碎片合并 + 深度图地面高度裁决 + 跨观察点地图合并。

目标：判断两者有没有其它代码逻辑上的冲突、能否移植到 aic26_ws_112。本轮只出评估报告，不改代码；若将来实施，倾向改进现有 `ebike_pano`，不新增 target。另明确：工作区 ebike 终端输出（违停数、倒伏数）作为裁判 `task=ebike` 的上报正文。

## 环境信息

- 车端：`mowen@192.168.43.15`（mDNS `mowen-JNE10.local`），N100 / 4 核 / 7.5 GB。
- aic26_ws_112：分支 `rollback/desktop-0321` @8f332e8，工作树脏（含未提交的 `ebike_pano` LazyYoloNet/conf 0.70 改动）。
- 相机：Orbbec DaBai DCW2 `2bc5:0561`，UVC 彩色 / Orbbec SDK 彩色+深度两条访问路径。
- aic26 相机节点：`src/aic26_cobot/scripts/camera_node.py`，UVC（OpenCV），1920×1080@30，突发 6 帧后释放设备、冷却 1 s；无深度话题。
- rc 深度链路：`roslaunch orbbec_camera dabai_dcw2.launch color_width:=1440 color_height:=1080 depth_registration:=true`，深度 640×480 uint16 mm，配准到彩色；SDK 会解绑 uvcvideo，退出需 `reattach_uvc()`。

## 报错记录

本轮无运行报错；以下为核实时发现的配置矛盾与风险点。

## 涉及文件

aic26_ws_112：

- `src/aic26_perception/scripts/ebike_pano_detector.py`：views=3、view_frames=6、frame_timeout=2.0、conf 0.70、MIN_CENTRE_Y=0.30、MIN_AREA_FRAC=0.0005、DUP_IOU=0.45；末视角 `stitch_and_crop` 后推理，失败回退最忙单视角。
- `src/aic26_perception/scripts/ebike_stitch.py`：OpenCV Stitcher（PANORAMA→SCANS→wave），裁黑边/coverage。
- `src/aic26_perception/scripts/ebike_dedup.py`：现成的像素→地面→地图投影（`project_pixel:106`、`project_box:148`），默认 K 1280×720、相机 `(0.02, 0, 0.2025)`、光学系 rpy `(-90,0,-90)`。
- `src/aic26_cobot/scripts/camera_node.py:60-68,260-268`：突发出帧与 IDLE/VIEW/FULL 模式。
- `src/aic26_description/urdf/cobot.xacro:15-17,126-137`：相机高 0.2025 m、俯仰 0。
- `src/aic26_decision/config/real_nostrafe_tasks.yaml:290-299`：panorama 任务，waypoint `ebike_pano`，views `[-0.5236, 0, 0.5236]`，announce=false。
- `src/aic26_decision/config/real_nostrafe_waypoints.yaml:137`：航点约 `(1.1, 0.6, yaw 90°)`。
- `src/aic26_decision/scripts/mission_controller.py:543-553,1580-1605,1621-1622`。
- `src/aic26_decision/scripts/mission_common.py:772-830`：终端/播报 ebike 句子。
- `src/aic26_decision/scripts/referee_bridge.py:52-53,132-142`：裁判 ebike 文字与选图。
- `src/aic26_perception/scripts/review_recorder.py:42-45`：DEFAULT_TARGETS 含 ebike_pano；frame_id 契约 `ebike_pano_view_<N>` / `ebike_pano_pano`。

rc：

- `tools/vision_worker.py`：`job_parking:759`、`park_zoom:627`、`park_boxes:661`、`burst_vote:698`、`park_stitch:845`、`parking_merge:915`、`_use_ground:164`、`h_cm` 计算 `:677`、阈值判断 `:679-682`。
- `tools/geom3d.py:14,41`：相机常量 0.262 m / 15.4°，合理性窗口 0.24–0.29 m / 11–20°。
- `config/vision.json` parking 段：zone_x `[-0.5,0.3]`、zone_y `[-0.2,1.99]`、stand_cm 11.0、lie_cm 10.9、stand_top_cm 8.0、lie_top_cm 7.0、max_top_cm 16.0、merge_m 0.065、free_space。
- `models/parking_bike_640_v2.onnx`：640，类别 fallen/upright，输出 `[1,6,8400]`，conf 0.37（zoom 0.45）。
- `station/server.py:62-70,290,449-456,708-713,802-803`：深度采集/写盘/驱动切换。

## 修改方案

本轮不实施。评估出的分级：

### A 档：可直接搬（换后端 + 适配接口）

1. 模型 `parking_bike_640_v2.onnx` → `src/aic26_perception/models/`（先验证 OpenVINO 能加载 `[1,6,8400]`）。
2. zoom 多尺度切片（rc `park_zoom`）搬进 `ebike_pano_detector.infer`，用 `LazyYoloNet`，临时改 conf 后恢复。
3. 3 帧 burst 投票（rc `burst_vote`）改为使用本视角已缓存的 6 帧，不依赖文件名。
4. 同视图碎片合并（rc `free_space`）：无深度时退化为图像接触 + 地图距离。
5. 类别映射 fallen/upright ↔ down/normal，保持 `total/down/normal/boxes/text` schema 不变。
6. 预计 1–2 人日改代码 + 1 人日离线回放 A/B。

### B 档：必须重标 / 重写

1. `h_cm` 公式与 stand 11.0 / lie 10.9 阈值：依赖相机高与 fy，需按 aic26 实拍重测。
2. 不要搬 `geom3d` 常量/窗口，改用 `ebike_dedup.py` 的投影；先校正其默认外参。
3. `zone_x/zone_y`：rc 场地绝对坐标，需在 aic26 地图上重标。
4. pose/地图投影：aic26 无 `index.jsonl`，改走 TF `map←camera_optical_frame`，回退 `map←base_footprint`+外参，再回退 AMCL。
5. 分辨率/内参：1920×1080 且未标定，需订阅 `camera_info`；精确投影前先做一次 RGB 标定。
6. 预计 0.5–1 人日标定 + 2–3 人日调参与复跑。

### C 档：阻断 / 不应移植

1. 深度地面链（`Ground/ground/fit/obj`）：aic26 无深度流；要用需改相机驱动或停车点临时切深度服务。
2. 跨圈 `parking_merge`（全局 `results` 下标、`VISION_PRELOAD`）：aic26 节点常驻整场，会跨圈污染；去重应在单次 panorama 的 3 视角内做。
3. rc 的文件名 / `index.jsonl` 协议：aic26 不落逐帧图与位姿。
4. 新增独立 target：触发 10+ 注册点与 Tk/裁判改动。
5. ultralytics 依赖：破坏 OpenVINO/LazyYoloNet 的内存与线程预算（16 常驻模型约 2.7 GB）。
6. 把倒伏做成独立裁判任务：裁判任务表固定 8 类，停车不单列。

### 深度可行性

- 同车同相机，rc 的 `orbbec_camera` 深度链路与标定可直接参考；两套栈轮流跑，不存在跨栈并发争设备。
- 三条路径：①换 aic26 相机驱动（失去突发省 CPU，30 fps 连续流对 N100 压力大）；②停车点临时切深度服务（停 camera_node → 起 SDK → 取帧 → 退出 → 重绑 UVC，约 10–30 s）；③不取深度（单目，风险最低）。
- 关键矛盾：aic26 URDF 写 0.2025 m / 0°，rc 11-24 用 60 张深度图实测 0.262 m / 俯 15.4°。同车同相机只能有一个真值，rc 深度实测更可信，URDF 很可能过时；这也说明 aic26 现有 `ebike_dedup` 默认外参可能偏。需现场量一次定值：若实物确为俯视，rc 的地面拟合可复用；若确为水平，则只能改简化高度判定。二选一。

## 结果与结论

- **语义重复**：两套都输出"站立/倒伏"。若并存会在同一停车区产出两套倒伏数，汇总、播报、裁判文字必有一处打架。
- **可移植的是检测后处理，不是几何**：模型、zoom、burst 投票、碎片合并、状态映射可搬；h_cm/阈值、geom3d 常量与窗口、zone、地图投影必须重标或重写；跨圈合并、深度地面链、文件协议不应搬。
- **推荐**：若实施，只改进现有 `ebike_pano`（A 档，单目无深度），target/任务/Tk/裁判口径全部不变；回退=还原 `ebike_pano_detector.py` 与 `ebike.launch`。完整深度方案属于相机架构改造，不建议在当前周期做。
- **裁判 ebike 上报口径现状已满足**：停车区完成后 `_emit_ebike_summary()` 用 `format_ebike_terminal_summary()` 生成终端句并 `rospy.loginfo` 写入 mission 日志；`referee_bridge._ebike_report()` 抓取以 `A街区`/`B街区` 开头且含 `违停` 的文本，替换"电动车"为配置词后作为 `task=ebike` 正文上传。句子已含"停车区内电动车正常 N 辆，倒伏 M 辆"。唯一缺口：三个视角都失败时终端句不含停车区半句，上报带 `停车区本圈未识别`，倒伏数缺失。
- **待现场确认**：相机实物高度/俯仰（URDF 与 rc 实测冲突）；aic26 停车区真值（正常/倒伏各几辆）；是否接受停车点切深度的 10–30 s 停摆。

## 相关记录

- [实车八类报告行与终端过滤](2026-11-25_实车八类报告行与终端过滤.md) —— ebike 终端句与报告行现状。
- [A/B 街社区人员与 ebike_park 分区统计](2026-11-23_AB街人员与ebike_park分区统计.md) —— 违停按街累计。
- [ebike 两站视觉去重与独立去重节点](2026-11-16_ebike两站视觉去重与独立节点.md) —— ebike_dedup 的投影与 TF 回退。
- [AIC26 实车相机与视觉检测稳定性排障](2026-11-15_AIC26实车相机与视觉检测稳定性排障.md) —— 相机与检测器稳定性。

## 变更记录

- 2026-11-28 21:24 首次整理
