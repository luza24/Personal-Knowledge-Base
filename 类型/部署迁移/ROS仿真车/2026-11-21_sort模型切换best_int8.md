---
title: 实车 sort 识别模型切换为 best_int8（QDQ int8 1280）
date: 2026-11-21
updated: 2026-11-24
project: ROS仿真车
type: 部署迁移
themes: [感知与视觉, 硬件与调试]
tags: [ros, yolo, openvino, perception, 真车]
status: 部分解决
---

# 实车 sort 识别模型切换为 best_int8（QDQ int8 1280）

> 整理日期：2026年11月24日 10:19
> 项目：ROS仿真车 ｜ 类型：部署迁移 ｜ 状态：部分解决 ｜ 主题：`感知与视觉` `硬件与调试` ｜ 标签：`ros` `yolo` `openvino` `perception` `真车`

## 背景 / 目标

用户提供 `C:\Users\luza\Downloads\best_int8.onnx`（27,060,362 B，md5 `9febc607476d9cde292f29ba06f7f3e0`，
2026-11-21 21:42），要求把实车 `aic26_ws_112` 垃圾分类（sort）任务使用的模型从 fp32 `sort.onnx`
换成该 int8 模型。目标：先核查类别 / 后处理 / 推理链兼容性，再统一切换所有加载入口，并保留 fp32 回退。

## 环境信息

- 实车：`192.168.1.15` / `mowen`，Ubuntu 20.04 + ROS Noetic，N100 CPU
- 工作区：`/home/mowen/aic26_ws_112`（`speed-tuning` 分支，无 git remote）
- 推理：OpenVINO 2024.4（CPU，2 线程）、onnxruntime 1.19
- 相机：Orbbec DaBai DCW2，检测时 1920×1080
- sort 协议：`conf 0.75`、`min_frames=2`、0.8 s 观测窗口，原 `sort_detect_timeout` 8 s

## 核查结果（旧 vs 新）

| 项 | 旧 `sort.onnx` | 新 `best_int8.onnx` |
|---|---|---|
| 类别 | 24 类 | 24 类，**名称与顺序完全一致** |
| 输入 | `[1,3,640,640]` fp32 | `[1,3,1280,1280]` fp32（图内 QDQ int8） |
| 输出 | `[1,28,8400]` | `[1,28,33600]`（4 box + 24 类，YOLOv8 同布局） |
| 量化 | 无 | QDQ（onnxruntime.quant，163 Quantize / 330 Dequantize） |
| 导出 | Ultralytics 8.4.164，2026-11-15 | Ultralytics 8.4.211，2026-11-21，imgsz 1280 |
| 单帧推理（2 线程） | 393 ms | **2008 ms**；4 线程 1274 ms；ORT 3679 ms |
| 真实帧检出 | 每帧仅 1–2 张卡片 | 每帧 4 张全检出（score 0.76–0.96） |

关键结论：

- `perception_common.YoloNet._resolve_size()` 会自动切 1280 letterbox，`_decode()` 兼容新输出形状，
  **检测代码无需修改**。
- OpenVINO 与 ORT 对新模型逐帧解码的类别 / 坐标一致，分数差 ≤0.04，QDQ 路径正确。
- 图内 DFL `Reshape` 锚点数硬编码，**不能 reshape 成 640**；要提速只能按 imgsz=640 重新导出。

## 涉及文件

车辆 `~/aic26_ws_112`：

- `src/aic26_perception/models/sort_qdq_int8.onnx`（新增，27,060,362 B）
- `src/aic26_perception/models/sort.onnx`（原样保留，44,782,475 B，作 fp32 回退）
- `src/aic26_bringup/launch/mission.launch`
- `src/aic26_perception/launch/sort.launch`
- `src/aic26_perception/scripts/sort_detector.py`
- `src/aic26_perception/scripts/detect_gui.py`
- `src/aic26_perception/models/README.md`
- `WORKSPACE_STATE.md`
- 以上改动文件均备份为 `*.bak_20261122_sort_int8`

## 修改方案

1. **模型部署**：scp 到 `/tmp/best_int8.onnx` → `src/aic26_perception/models/sort_qdq_int8.onnx`，
   md5 与源文件一致（`9febc607…`）；旧 `sort.onnx` 原样保留。
2. **`mission.launch`**：`sort_model` 指向新模型；`sort_detect_timeout` 8→12 s；更新“44 MB fp32”过期注释。
3. **`sort.launch`**：默认 `model_path` → 新文件；`input_size` 640→1280（仅为日志清晰，回退旧模型时
   `YoloNet` 仍会自动降回 640）。
4. **`sort_detector.py`**：`default_model_path()` 返回 `sort_qdq_int8.onnx`。
5. **`detect_gui.py`**：`TARGETS['sort']['model']` 由 `'sort.onnx'` 改为 `'sort_qdq_int8.onnx'`
   （`SORT_CLASSES` 与 `conf 0.75` 不变），保证 GUI 独立检测入口也走新模型。
6. **文档**：`models/README.md` sort 段注明 int8/1280/类别顺序不变/旧文件可回退；
   `WORKSPACE_STATE.md` 模型表与变更记录同步。

未修改 `perception_common.py`、`mission_controller.py`、任务 YAML、相机逻辑。

## 结果与结论

- 冒烟测试（实代码路径 `YoloNet`，不启 ROS）通过：
  `using model input size 1280x1280 (was 640)`、
  `loaded sort_qdq_int8.onnx with openvino (CPU, 2 threads)`，review 图 `dets=4`
  （class 14 score 0.913、class 23 score 0.892 等）；两个脚本 `py_compile` 通过。
- `roslaunch --dump-params` 确认 `sort_detect_timeout: 12.0`、`sort_detector/conf: 0.75`。
- 代价：单帧 2.0 s（旧 0.39 s），sort 请求预计 4–6 s，整站停靠 +3–5 s。
- 回退：`sort_model:=$(find aic26_perception)/models/sort.onnx` 启动，或用 `.bak_20261122_sort_int8` 恢复。
- **待实车联调**：确认 conf 0.75 下 4 张卡片无双检 / 漏检、整站时长可接受；必要时把 `conf` 降到 0.6。
- 可选 `AIC26_OV_THREADS=4` 把单帧降到 ~1.27 s，但会影响所有检测节点，暂不启用。

## 相关记录

- [垃圾投放正确性判定与 sort 切换 1920×1080](2026-11-20_垃圾投放正确性判定与sort1080p切换.md)
- [AIC26 垃圾分类 sort 检测接入与置信度调优](2026-11-15_AIC26垃圾分类sort检测接入与置信度调优.md)
- [OpenVINO int8 视觉检测失效修复（aic26_ws_112）](2026-11-04_OpenVINO_int8检测失效修复.md)

## 变更记录

- 2026-11-21 21:42 best_int8.onnx 导出完成，开始实车兼容性核查（事件）
- 2026-11-22 14:06 模型替换、launch/脚本/文档更新与冒烟验证完成（事件）
- 2026-11-24 10:19 首次整理（回补 2026-11-21 会话）
