---
title: aic26_perception 容器版与 GitHub visual-slam 分支对比
date: 2026-10-08
updated: 2026-10-08
project: ROS仿真车
type: 方案选型
themes: [感知与视觉, 环境与工具链]
tags: [ros, perception, git]
status: 已解决
---

# aic26_perception 容器版与 GitHub visual-slam 分支对比

> 整理日期：2026年10月8日 21:05

## 环境信息
- 运行环境：Docker 容器 `ros_noetic_gazebo`（镜像 `osrf/ros:noetic-desktop-full`）
- 容器内源码路径：`/root/catkin_ws/src/aic26_perception`（完整 git 仓库，根目录 `/root/catkin_ws`）
- 对比的两个版本：
  - **容器当前版**：分支 `feature/nav-slam`，HEAD `5f890fb`
  - **GitHub visual-slam 版**：`origin/feature/visual-slam`，HEAD `7a86b34`（另有本地 `feature/visual-slam` 多一条 `34882a9` 模型更新提交）
- 说明：本次对比直接使用容器内已拉取的 git 引用完成，无需联网（宿主机 git 访问 GitHub 被重置，`WebFetch` 亦被拦截）。

## 报错记录
- 本次为纯对比调查，无报错。

## 视觉模型对比结论
两个权重文件**字节级完全一致**：

| 文件 | blob hash | 结论 |
|---|---|---|
| `weights/best.pt` | `8a7944f858d570260b2759a13039c71e6bed5bd9` | 完全一致 |
| `weights/tl_state_det.pt` | `6bcc9dd50061943b4d077208e8c17768a1ce3a7c` | 完全一致 |

- 唯一差异：`best.pt` 文件权限位（容器 `100755` 可执行 vs GitHub `100644`），不影响内容。
- 即 GitHub visual-slam 分支上「更新视觉模型 best.pt（train-merged 仿真域重训练）」提交（`34882a9`）的成果，**容器当前版本已包含**，模型无区别。

## 代码对比结论
容器 `feature/nav-slam` 分支比 GitHub `visual-slam` 分支**领先 5 个提交**，属于「升级版」，差异如下：

1. **抽取 `vision_utils.py` 公共模块**（新增 257 行）
   - 把两个节点重复的常量（`STAGE1_NAMES`/`COLORS`/`KNOWN_PLATES`/`MATCH_THRESHOLD`）与函数（`tl_state_hsv`/车牌 OCR/`match_plate`/`nms_boxes`/图像编解码）集中到一个模块。
   - visual-slam 版是每个脚本各自内联一份重复代码。
2. **`yolo_save_node.py` 重写**（264→42 行）
   - visual-slam 版：持续检测+保存节点，自加载模型、订阅 `/patrol/rotating` 与 `/yolo/save_once`，旋转/任务期间每秒推理并保存。
   - 容器版：改为纯存图节点，不加载模型、不推理，只订阅 `/yolo/annotated_image` 落盘（与 detector 分工，避免重复加载模型）。
3. **车牌 OCR 算法增强**
   - `_ocr_crop` → `_ocr_items`：按纵向中心聚类成行、过滤「number_plate」标签等杂讯行。
   - `read_plate(crop_img)` → `read_plate(frame, box)`：整帧+检测框，框外扩 35% 重试、放大到 768px 高、省份汉字开头且≥6 位校验、框贴边（被截断）直接放弃。
4. **新增车牌模糊匹配**
   - `_edit_distance`（Levenshtein）/`_similarity`/`match_plate`，已知车牌库 `KNOWN_PLATES`（鄂D7B5Q2 / 苏APL12A / 苏AB8Q62），阈值 0.85。
   - 检测结果新增 `matched_plates` 字段（含 raw/plate/score）。
5. **新增 YOLO 预热 + RapidOCR 预加载**
   - 节点启动时提前跑一次空图推理（避免 torch 首次算子初始化卡顿）并预加载 OCR 三个 onnx 模型（把 3~6s 冷启动提前）。
6. **新增红绿灯动态开关**
   - `~detect_traffic_light` 参数，patrol 侧可在任务点/旋转点关闭省时。
7. **标注图增强**
   - 检测框上追加置信度 + 车牌匹配结果（`匹配车牌(相似度)` / `未命中`）。

launch 文件、`CMakeLists.txt`、`package.xml` 仅有换行符（CRLF vs LF）与权限位差异，无实质内容变化。

## 涉及文件
- `src/aic26_perception/scripts/vision_utils.py`（新增，容器独有）
- `src/aic26_perception/scripts/yolo_detector.py`（重构，引用 vision_utils）
- `src/aic26_perception/scripts/yolo_save_node.py`（重写为纯存图）
- `src/aic26_perception/scripts/test_image.py`、`test_plate.py`（OCR 算法同步增强）
- `src/aic26_perception/weights/best.pt`、`tl_state_det.pt`（模型一致）
- `launch/`、`CMakeLists.txt`、`package.xml`（仅换行符/权限位差异）

## 结果与结论
- 视觉模型：**无区别**，两分支权重字节级一致。
- 代码：**容器版是 GitHub visual-slam 的升级版**，多出公共模块抽取、存图节点瘦身、车牌 OCR 增强、车牌模糊匹配、模型预热/OCR 预加载、红绿灯动态开关等。
- 若需与远程 visual-slam 对齐，可认为容器已「超前」，无需回退。

## 变更记录
- 2026-10-08 21:05 首次整理
