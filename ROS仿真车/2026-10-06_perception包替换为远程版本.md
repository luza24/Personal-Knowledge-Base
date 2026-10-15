---
title: 容器 aic26_perception 包替换为远程 feature/nav-slam 版本
date: 2026-10-06
updated: 2026-10-06
project: ROS仿真车
status: 已解决
tags: [部署迁移, ros, docker, git, perception]
---

# 容器 aic26_perception 包替换为远程 feature/nav-slam 版本

> 整理日期：2026年10月6日 10:50

## 环境信息
- 宿主：Windows 11 家庭中文版 + Docker Desktop（Docker 29.6.1，Git Bash 环境）
- 容器：`ros_noetic_gazebo`（镜像 `osrf/ros:noetic-desktop-full`），ROS Noetic 1.17.4
- 工作区：`/root/catkin_ws`（构建工具 `catkin_make`，纯 Python 包）
- 视觉运行环境：独立 venv `/root/ylovenv`（cv2 5.0、numpy 1.24.4、torch 2.4.1、ultralytics 8.4.166、rapidocr_onnxruntime，已通过 `ros.pth` 接入 ROS）
- 远程仓库：`https://github.com/Govis-Ink/aic26.git`，分支 `feature/nav-slam`

## 报错记录
- **catkin_make 同名包冲突**：
  ```
  Multiple packages found with the same name "aic26_perception":
  - aic26_perception
  - aic26_perception.bak_a619697
  ```
  触发条件：备份时用 `mv aic26_perception aic26_perception.bak_a619697`，备份目录仍留在 `src/` 树内，`catkin_make` 会扫描 `src/` 下所有子目录并把 `.bak` 目录当成第二个同名包。
  **解决**：把备份目录移出 `src/` 树（移到 `/root/`），再重新 `catkin_make`。

## 涉及文件
- `/root/catkin_ws/src/aic26_perception/`：整体替换为远程 HEAD 版本
  - `scripts/yolo_detector.py`、`scripts/yolo_save_node.py`、`scripts/test_image.py`、`scripts/test_plate.py`（4 个脚本有实际差异）
  - `CMakeLists.txt`、`package.xml`、`launch/*.launch`、`weights/*.pt`（与容器原版一致，md5 相同）
- `/root/aic26_perception.bak_a619697/`：替换前的原包备份（移出 src 树，可回滚）

## 修改方案
1. **git 对比定位版本**：
   - 容器原包逐字节一致于远程 commit `a619697`（"feat: patrol 巡逻重构与旋转/检测修复"，CRLF 去重后全部文件匹配），即容器无本地改动，是干净检出。
   - 远程 `feature/nav-slam` HEAD = `c2ad38a`（"vision: 改进车牌 OCR 识别并修复模型路径"），其父提交正是 `a619697`，且该提交**只改了 4 个脚本**。
   - 结论：容器只落后一个提交，替换 = 纯升级、无冲突。
2. **备份**：`mv /root/catkin_ws/src/aic26_perception /root/aic26_perception.bak_a619697`（关键点：备份目录必须移出 `src/`，否则 `catkin_make` 报同名包冲突）。
3. **替换**：宿主浅克隆远程分支后 `docker cp src/aic26_perception/. ros_noetic_gazebo:/root/catkin_ws/src/aic26_perception/`。
4. **清理 + 重建**：`rm -rf scripts/__pycache__`；`catkin_make --pkg aic26_perception`（成功，devel 安装已刷新）。
5. **核心差异**（`c2ad38a` vs `a619697`）：
   - 车牌 OCR 重构：旧 `_ocr_items/_clean_plate/_plate_text/read_plate(frame, box)`（多档拉伸重试 + LANZCOS4 + 放大 768）→ 新 `_ocr_crop/read_plate(crop_img)`（单次裁剪 + INTER_CUBIC + 放大 384）。
   - `test_image.py`/`test_plate.py` 模型路径由硬编码 `~/catkin_ws/src/aic26_perception/weights/best.pt` 改为相对 `../weights/best.pt`。

## 结果与结论
- 替换成功，验证全部通过：
  - 4 个脚本 `py_compile` 通过。
  - 权重相对路径 `../weights/best.pt` 正确解析。
  - 单图离线检测跑通：car1 图识别出 `car=1, number_plate=1, plates=['苏AEB8Q62']`。
  - `roslaunch aic26_perception yolo_detector.launch` 正常启动，加载权重、`/yolo/detect` 服务就绪。
- 备注：
  - `/yolo/detect` 服务 JSON 接口未变，不影响 `aic26_navigation` / patrol 脚本等其它包。
  - 新的 `test_image.py`/`test_plate.py` 需在 `scripts/` 目录下运行（模型路径是相对 `../weights/`），`roslaunch` 不受影响。
  - 备份 `/root/aic26_perception.bak_a619697` 建议稳定运行一段时间后再删。

## 变更记录
- 2026-10-06 10:50 首次整理
