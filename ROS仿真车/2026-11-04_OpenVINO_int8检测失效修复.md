---
title: OpenVINO int8 视觉检测失效修复（aic26_ws_112）
date: "2026-11-04"
updated: "2026-11-05"
project: ROS仿真车
tags: [报错排查, 视觉, yolo, perception, ros]
status: 部分解决
---

# OpenVINO int8 视觉检测失效修复（aic26_ws_112）

> 整理日期：2026年11月04日 23:53
> 项目：ROS仿真车 ｜ 状态：部分解决 ｜ 标签：`报错排查` `视觉` `yolo` `perception` `ros`

## 背景 / 目标

`aic26_ws_112` 仿真中 car / people / traffic_light 三个检测器几乎不出结果，而同样的三个 ONNX 模型在 `aic26_ws` 工作区正常。目标是找出两工作区视觉检测实现与模型调用方式的差异，定位 112 失效根因、修复并验证，最后整理成可交给 Astra 独立审查的源码包与文档。

## 环境信息

- 车端 Ubuntu + ROS1 Noetic（catkin），Python 3.8.10
- OpenVINO 2024.4.0-16579-c3152d32c9c-releases/2024/4
- onnxruntime 1.19.2，OpenCV 4.2.0，numpy 1.24.4，easyocr 1.7.2
- 无头仿真：Windows 本机 VcXsrv（`:0 -ac`）提供 X；软件渲染下 `/camera/image_raw` 约 2.6 Hz
- 工作区：`/home/mowen/aic26_ws_112`（分支 `speed-tuning`，HEAD `9089933`），参考 `/home/mowen/aic26_ws`
- 推理后端优先级：OpenVINO -> onnxruntime -> cv2.dnn

## 报错记录

- 现象：people 任务长期 `total: 0`；traffic_light 长期 `UNKNOWN`；car 偶发 1 个低分框，且跨帧不稳定。
- 两工作区 3 个 ONNX 的 md5 完全一致（`sim_*` 只是改名），排除模型文件差异。
- 启动日志正常：`loaded *_int8.onnx with openvino (CPU, 2 threads)`，OpenVINO 加载、编译、推理均不报错，因此不是“加载失败后未回退”。
- 同帧原始类别分数对比：
  - `core.read_model()`：car 最高 0.355~0.574，traffic 0.236~0.258，people 0.003~0.008；
  - `ov.convert_model()`：car 0.977，traffic 0.964，people 0.970。

## 涉及文件

- `/home/mowen/aic26_ws_112/src/aic26_perception/scripts/perception_common.py`（改）——三个检测器共用的 `YoloNet`
- `/home/mowen/aic26_ws/src/aic26_perception/scripts/perception_common.py`（参考实现，原样同步到 112）
- 112 其余未提交改动：`car_detector.py`（OCR/session）、`mission_controller.py`（`detect_wait=1.0`）、launch/配置等，均早于本次修复，与根因无关
- 备份：`scripts/perception_common.py.bak_20261104_211655_openvino`（md5 `dfdb47d9...`，与 HEAD 完全相同）
- 桌面审查包：`C:\Users\luza\Desktop\AIC26_视觉检测修复审查包_20261104\`
  - `code/`（112 工作树文本源码 185 个文件，约 600 KB）、`before/`（关键文件 HEAD 版本 11 个）
  - `git/`（HEAD/status/diffstat/完整 diff/perception diff/log）、`evidence/`（环境、对比实验、节点注入、任务日志）
  - 主文档 `问题与解决_OpenVINO_int8视觉检测_20261104.md`、`Astra_视觉检测修复审查提示词_20261104.md`、`README.md`、`MANIFEST.txt`
  - 同名 zip 与桌面根目录两份文档副本

## 修改方案

根因：OpenVINO 2024.4.0 用 `core.read_model()` 读取这批 int8 ONNX 时，类别分数被抹平（输出形状合法、无异常）；`YoloNet._load()` 又优先 OpenVINO，加载“成功”后 onnxruntime 永不回退，于是三个检测器同时**静默失效**。

最小修复（只改 `perception_common.py`）：

```python
# core.read_model() drops the class scores of these int8 ONNX
# graphs; convert_model() runs the full frontend and is correct.
if path.lower().endswith('.xml'):
    model = core.read_model(path)
else:
    model = ov.convert_model(path)
```

配套改动：

- 推理输入按张量对象绑定：`self.ov_request.infer({self.ov_input: blob})`；
- 输出 `np.array(..., copy=True)`，避免缓冲区复用；
- `_decode` 增加 `expected = 4 + self.num_classes` 通道校验/截断。

模型命名（`sim_*`/`real_*`）、`backend`/`fallback_path`/`ov_threads` 等属两工作区的整理差异，不是因果项。

## 结果与结论

- 修复已同步到 112 工作树；**未提交 git**。
- 离线同帧端到端：旧代码 `car 1 (0.574) / traffic 0 / people 0` → 新代码 `car 2 (0.977) / traffic 3 (0.964) / people 4 (0.970)`。
- ROS 节点注入已知帧：people 4；car 消息 1/2；traffic 状态 GREEN。
- 完整无头仿真任务：17 个任务全部跑完，`traffic_light_1`、`traffic_light_2` 均判 GREEN。
- 审查包已校验：zip 217 个条目，无 ONNX/图片/`build`/`devel`/`.git`/`*.bak_*`；MANIFEST 中 210 个文件 SHA256 全部匹配；`git/diff.patch` md5 与车端 `git diff` 一致（`62e17677...`）。

遗留问题（已写入主文档，待 Astra 排优先级）：

1. P1 车辆结果被 OCR/session 吞掉：`car_detector.on_image()` 先 `wait_for_ocr()`（最长 8 s）再 `publish_result()`，而任务只等 `detect_wait=1.0 s` 就发 `stop`；日志出现三次 `discarding stale OCR result (session 15/17/19, current 16/18/20)`。
2. P2 people 站点为 0：1.0 s 检测窗 + 无头软件渲染相机仅约 2.6 Hz；注入已知帧立即出 4。
3. P3 `cv2.dnn` 回退分支缺 `self.net.setInput(blob)`（仅 OpenVINO/onnxruntime 都不可用时触发，两工作区共有）。
4. 信息项：`results/*.json` 只在 `gui:=true` 的 `ResultViewer` 中写，无头运行不更新属设计行为。

## 2026-11-05 后续

在 `aic26_ws_112` 完成 v2 请求协议、people 单帧众数、car 异步 OCR/预加载、traffic event 和 mission 并行触发后，本文遗留的 P1（car OCR 被 session 吞掉）和 P2（people 1 秒窗口 0 人）已处理：

- people 改为第一帧后观察窗口，并在最新版本改为单帧众数，不再跨帧并集；
- car EasyOCR 后台预加载，图像回调不再 `join()`，最终结果只封存一次，无车/empty/timeout 都有明确结果；
- 仿真全程和实车 18 任务均跑通；
- 实车视觉准确率仍不足：people 多数站点为 0，car OCR 多为 empty/timeout，traffic 未在 10 秒内判定绿灯。

完整过程见 [AIC26 视觉检测 v2 与实车 PC 语音链路验证](2026-11-05_AIC26视觉v2实车链路.md)。

## 相关记录

- [aic26_ws_speech 全工作区梳理与语音链路风险](2026-10-25_aic26_ws_speech梳理与风险.md)
  —— 同一套三检测器 + `perception_common` 的整体梳理；本记录补上了 OpenVINO 层面的失效根因。
- [aic1023 代码审查：转向、OCR 与导航超时的关键缺陷](2026-10-25_aic1023代码审查与关键缺陷.md)
  —— 早期版本同类视觉链路的审查，可作为 OpenCV DNN / OpenVINO / onnxruntime 后端对照。
- [aic26 双工作区导航对比与备份](2026-11-01_aic26双工作区导航对比与备份.md)
  —— 112 与主工作区并存的背景，本次修复只在 112 的 perception 包内做最小改动。

## 变更记录

- 2026-11-04 23:53 首次整理
- 2026-11-05 22:59 补充：v2 视觉链路完成并实车跑通；P1/P2 已处理，新增实车准确率遗留
