---
title: 实车车牌 OCR 从 EasyOCR 换成 PaddleOCR
date: 2026-11-15
updated: 2026-11-19
project: ROS仿真车
type: 功能开发
themes: [感知与视觉]
tags: [ocr, perception, ros, 真车]
status: 部分解决
---

# 实车车牌 OCR 从 EasyOCR 换成 PaddleOCR

> 整理日期：2026年11月15日 19:21
> 项目：ROS仿真车 ｜ 类型：功能开发 ｜ 状态：部分解决 ｜ 主题：`感知与视觉` ｜ 标签：`ocr` `perception` `ros` `真车`

## 背景 / 目标

实车 `~/aic26_ws_112` 的车牌识别一直用 EasyOCR，真实车牌基本读不对。目标是把 `car_detector.py` 与独立调试工具 `detect_gui.py` 的 OCR 彻底换成 PaddleOCR，同时保持现有 JSON 契约、状态值、预加载和 respawn 生命周期不变。用户确认：直接替换（不做双后端）、两个入口都改、保持现有后处理（tidy + 字符合集 + 长度≥4 + 按长度/置信度排序）。

## 环境信息

- 车端 `mowen@192.168.1.15`，`/home/mowen/aic26_ws_112`，Ubuntu 20.04 / ROS Noetic / Python 3.8.10 / Intel N100。
- `paddleocr 2.7.0.3` + `paddlepaddle 3.0.0` 已安装；PP-OCRv4 det/rec/cls 模型已缓存在 `~/.paddleocr/whl`（约 18 MB），可完全离线。
- EasyOCR 1.7.2 保留安装但代码不再引用。
- 实测 PaddleOCR 构造 0.78 s、单次推理 0.1–0.3 s、峰值 RSS 约 575 MB（EasyOCR 约 1085 MB）。

## 报错记录

改造前后在相同车牌上的对照（tidy 后）：

| 测试样本 | 真值 | PaddleOCR | EasyOCR（原） |
|---|---|---|---|
| car_1_roi.jpg（仿真） | 鄂D7B5Q2 | `鄂D7B5Q2` 0.982 | `785021` 0.594 |
| car_2_roi.jpg（仿真） | 苏A6Q9T2 | `苏A6Q9T2` 0.997 | `苏4609121` 0.331 |
| car_3_roi.jpg（仿真） | 苏A7K3M8 | `苏A7K3M8` 0.938 | `苏7K318` 0.692 |
| 20261109 g19 真实车牌 | 蒙NZA772 | `蒙NZA772` 0.989（3×） | `GNZ4772` 0.225 |
| 20261109 g20 真实车牌 | 苏Q107V9 | `苏Q107V9` 0.996（3×） | `60107V91` 0.371 |
| 20261109 g22 真实车牌 | 苏RMV255 | `苏RMV255` 0.999（3×） | `HRM2551` 0.599 |

生产分辨率 1280×720 端到端（car_1/2/3.jpg：车辆检测→固定 ROI→OCR）PaddleOCR **3/3**（0.97/1.00/1.00）；EasyOCR 0/3。

60 帧 20261114 连拍 A/B：两边完全一致（44 帧 empty、16 帧无车 idle），因为那批帧本身没有可读车牌——只能证明无回归。

## 涉及文件

- `src/aic26_perception/scripts/plate_ocr.py`（**新增**）：共享 OCR 层。
- `src/aic26_perception/scripts/car_detector.py`：EasyOCR → `PlateOcrReader`，新增 `~ocr_model_root`、`~ocr_warmup`、`~ocr_retry_interval`。
- `src/aic26_perception/scripts/detect_gui.py`：同步换引擎，新增 `--ocr-threads/--ocr-scale/--ocr-binary/--ocr-min-chars/--plate-pad`。
- `src/aic26_perception/launch/car.launch`、`CMakeLists.txt`、`package.xml`。
- 文档：`README.md`、`src/aic26_perception/models/README.md`、`WORKSPACE_STATE.md`。
- 备份：上述文件均留 `.bak_20261115_paddle`；改前 git 快照 `6b8dd52`，tag `pre-paddle-ocr`。

## 修改方案

- `plate_ocr.py` 统一提供：`preprocess_plate`（灰度→GaussianBlur→可选二值化→默认 3× INTER_CUBIC）、`tidy_plate`、字符白名单过滤、`crop_car_plate`/`crop_frame_plate`、`PlateOcrReader`。
- 引擎延迟导入：`load()` 内先设 OMP/MKL 线程变量，再 `import paddleocr`。
- 离线兜底：构造前校验 det/rec/cls 三个目录的 `inference.pdmodel` + `inference.pdiparams`；把 `paddleocr.maybe_download` 打补丁为直接抛错，避免其无超时 HTTP 请求挂死 preload 线程（2.7.0.3 即使 `use_angle_cls=False` 也会解析 cls 模型）。
- `PaddleOCR(lang='ch', ocr_version='PP-OCRv4', use_gpu=False, use_angle_cls=False, show_log=False, drop_score=0.0, det_limit_side_len=960, det_limit_type='max', enable_mkldnn=False)`；固定 `ocr.ocr(img, cls=False)`，只传单个 ndarray。
- `drop_score=0.0` 等价于原来“不设置信度阈值”，仍由长度/字符合集过滤。
- 预热：load 末尾用合成 `A12345` 图跑一次 det+rec，保证第一次真实请求落在 mission 5 s `car_detect_timeout` 内。
- `recognize()` 异常统一返回 `('', 0.0, 'failed')`；构造失败捕获 `(Exception, SystemExit)`（paddleocr 失败会 `sys.exit(0)`，daemon 线程会吞掉）。
- 后处理保持原样：`tidy → allowlist → len>=4 → 按 (长度, 置信度) 降序取第一`。白名单补上原来漏掉的“浙”。
- `detect_gui --plate-pad` 默认改为 **0.05**，与生产节点一致；原来的 0 外扩太紧，PaddleOCR 检测框出不来。

## 结果与结论

- ROS 私有 master 完整请求链路：`鄂D7B5Q2 / 0.975 / ocr_status=ok / complete=true`；预加载约 3.5 s，`ocr_ready=ready`。
- fail-soft（模型目录缺失）：`ready=false / status=failed`，reason 列出缺失路径，**不下载、不挂起、节点保持注册**；装载失败 30 s 退避。
- `catkin_make --pkg aic26_perception` 通过，devel relay 生成。
- 状态“部分解决”：**尚未跑完实车全任务**，三个 car 站点的最终验证待做。
- 注意：在 roslaunch + 私有 master 测试时遇到工作区已记录的“节点启动约 5 秒被外部 SIGTERM”问题，直接运行节点则稳定；实车 bringup 时需观察 `/perception/car/ocr_ready` 和 respawn。
- 回滚：`cp -p <file>.bak_20261115_paddle <file>`，或 `git reset --hard pre-paddle-ocr`（会丢弃迁移后改动）。

## 2026-11-19 detect_gui 扩展：检测图按 target 归档 + building/car 通用 OCR

- **输出目录**：`detect_gui.py` 的保存路径改为 `<工作区>/detect/<target>/`，标注 jpg 与结果 json
  同目录；`--save-dir` 仍表示根目录，GUI 的 `s`/保存按钮与 headless 共用 `save_outputs()`。
- **OCR 预加载**：`car` 和 `building` 在脚本启动时即加载工作区 PaddleOCR（`plate_ocr.py`，
  离线 PP-OCRv4），终端打印 `loading paddleocr` / `paddleocr ready`。
- **car**：保留原车牌 ROI 识别，GUI 和 headless 都新增终端输出
  `car plate OCR: <text> (<conf>, <status>)`。
- **building**：新增 `PlateOcrReader.recognize_text()`（无车牌灰度预处理、无字符白名单，
  直接跑通用中文 OCR），对每个 building 检测框外扩 5% 裁剪后识别；
  终端逐行打印 `building OCR [box N]: 文本 (置信度)`，结果 JSON 增加
  `ocr: [{box, text, conf}]`，GUI 右侧面板显示识别文字。
- **验证**：`--target building --source results/building.jpg --headless` 输出
  `building OCR [box 0]: 极光大厦 (1.00)`，JSON `ocr` 正确；`--target car` 正常预加载，
  在 `results/car.jpg` 上模型未检出车辆（非回归）；`py_compile` 通过。
- **备份**：`detect_gui.py.bak_20261119_detectdir`、`detect_gui.py.bak_20261119_building_ocr`、
  `plate_ocr.py.bak_20261119_text_ocr`。

## 相关记录

- [AIC26 视觉检测 v2 与实车 PC 语音链路验证](2026-11-05_AIC26视觉v2实车链路.md) —— 原 EasyOCR 车牌链路与检测协议背景。
- [OpenVINO int8 视觉检测失效修复（aic26_ws_112）](2026-11-04_OpenVINO_int8检测失效修复.md) —— 同一工作区的检测推理环境。
- [实车 traffic_2_north 门口卡顿：航点格角与退避进墙](2026-11-15_实车门口卡顿航点格角修复.md) —— 同日导航侧改动。

## 变更记录

- 2026-11-15 19:21 首次整理
- 2026-11-19 22:34 补充：detect_gui 检测图按 target 归档；building 通用 OCR、car 车牌终端输出；启动时预加载 PaddleOCR
