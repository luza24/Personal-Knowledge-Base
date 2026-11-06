---
title: Docker 容器 ROS 代码推送到 GitHub 分支方法
date: 2026-09-30
updated: 2026-09-30
project: 开发环境
type: 环境配置
themes: [环境与工具链]
tags: [docker, git]
status: 已解决
---

# Docker 容器 ROS 代码推送到 GitHub 分支方法

> 整理日期：2026年9月30日 15:44

## 环境信息
- 操作系统：Windows 11 Home China 10.0.26200，Docker Desktop
- 容器：`ros_noetic_gazebo`（镜像 `osrf/ros:noetic-desktop-full`）
- 工作空间：容器内 `/root/catkin_ws`（git 仓库根），源码在 `/root/catkin_ws/src`
- 远端仓库：`git@github.com:Govis-Ink/aic26.git`，目标分支 `feature/nav-slam`
- SSH 认证用户：`luza24`；git 提交作者：`luza2007 <luza2007@qq.com>`

## 报错记录
本次无报错。但有几个若不处理会导致推送失败/仓库损坏的坑：
- `main` 与 `feature/nav-slam` 是**无关历史**（`git merge-base` 无输出），直接 merge 会失败，只能文件级同步。
- `src/gazebo_ros_model_color` 是**嵌套 git 仓库**，直接 add 会变成失效的 submodule 引用。
- `src/aic26_perception/scripts/__pycache__/` 及 `.pyc` 是构建产物，不能入库。
- `src/aic26_perception/weights/best.pt` 有 22MB，需确认未被 .gitignore 误忽略。

## 涉及文件
- 本次推送内容（feature 工作区 `/tmp/nav-slam-wt/src/`）：
  - `aic26_gazebo/`：新增 `scripts/traffic_light_controller_node.py`、3 个镜头模型 `traffic_light_{red,green,yellow}_lens`，改 `worlds/aic26-1.world`、`launch/aic26-1.launch`、`package.xml`
  - `aic26_navigation/`：`patrol_green_test.py` 改为调用 `/yolo/detect` 服务，新增 `patrol_move_base.py`
  - `aic26_perception/`：新增 YOLO 检测 `launch/`、`scripts/yolo_detector.py`、`weights/best.pt`
  - `aic26_utils/`：删除旧 C++ 控制器 `src/traffic_light_controller.cpp`
  - `gazebo_ros_model_color/`（第三方包）、`src/README.md`

## 修改方案（推送方法）

### 0. 前置检查
```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
docker exec ros_noetic_gazebo bash -lc 'ssh -T git@github.com'   # 确认 SSH 已认证
```

### 1. 摸清仓库与分支结构
```bash
docker exec ros_noetic_gazebo bash -lc 'cd ~/catkin_ws && \
  git rev-parse --show-toplevel; git branch -a; git worktree list; \
  git merge-base main feature/nav-slam'
```
若 `merge-base` 无输出 → 无关历史 → 用文件同步而非 merge。

### 2. rsync 同步 src 到目标分支 worktree
```bash
docker exec ros_noetic_gazebo bash -lc '
  rsync -a --delete \
    --exclude ".git/" --exclude "__pycache__/" --exclude "*.pyc" \
    /root/catkin_ws/src/ /tmp/nav-slam-wt/src/'
```
三个 `--exclude` 分别对应：嵌套 git、`__pycache__`、`.pyc`；`--delete` 让废弃文件（旧 C++ 控制器）正确删除。

### 3. 分两次提交（核心改动单独用指定说明）
```bash
docker exec ros_noetic_gazebo bash -lc 'cd /tmp/nav-slam-wt && \
  git add src/aic26_gazebo && git commit -m "解决红绿灯控制问题" && \
  git add -A src && git commit -m "同步感知/导航与红绿灯控制器迁移"'
```

### 4. 推送并验证
```bash
docker exec ros_noetic_gazebo bash -lc 'cd /tmp/nav-slam-wt && \
  git push origin feature/nav-slam && \
  git rev-parse HEAD origin/feature/nav-slam'
```
两行哈希一致即成功。

## 结果与结论
推送成功。远端 `origin/feature/nav-slam` 更新为 `f826254`，本地 HEAD 与远端哈希一致。
共 2 个提交：
- `52eab25` 解决红绿灯控制问题（`aic26_gazebo`）
- `f826254` 同步感知/导航与红绿灯控制器迁移（其余全部）

## 变更记录
- 2026-09-30 15:44 首次整理
