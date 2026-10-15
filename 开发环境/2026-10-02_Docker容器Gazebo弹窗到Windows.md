---
title: Docker 容器 Gazebo 不弹窗到 Windows 桌面（显示后端排查 + VcXsrv 直连）
date: 2026-10-02
updated: 2026-10-02
project: 开发环境
status: 已解决
tags: [报错排查, docker, gazebo, wsl2]
---

# Docker 容器 Gazebo 不弹窗到 Windows 桌面（显示后端排查 + VcXsrv 直连）

> 整理日期：2026年10月2日 11:21

## 环境信息
- 操作系统：Windows 11 Home China（WSL2 + Docker Desktop）
- WSL 发行版：Ubuntu（Running，v2）、docker-desktop、Ubuntu-20.04（Stopped）
- 容器：`ros_noetic_gazebo`，镜像 `osrf/ros:noetic-desktop-full`，启动命令 `bash`、入口 `/ros_entrypoint.sh`，网络模式 **host**
- 容器唯一挂载：`bind /tmp/.X11-unix -> /tmp/.X11-unix`（rw）
- 显示相关工具（容器内）：Xvfb、x11vnc、fluxbox、socat、websockify/noVNC、xdpyinfo
- 图形通道：WSLg（WSL 内 `/tmp/.X11-unix/X0` 与 `/mnt/wslg/.X11-unix/X0` 为同一 socket，owner=luza，inode=13 dev=90）

## 报错记录
1. `roslaunch aic26_gazebo aic26-1.launch` 报：
   `RLException: [aic26-1.launch] is neither a launch file in package [aic26_gazebo] nor ...`
   —— 实际文件存在，根因是**未 source 工作区**（`source ~/catkin_ws/devel/setup.bash` 后 `rospack find` 正常）。
2. **Gazebo 不弹窗到 Windows 桌面**（核心问题）：`gzserver`/`gzclient` 进程都在跑、`spawn_armbot` 显示 "process has finished cleanly"（小车已加载），但窗口不出现在 Windows。`gzclient` CPU 高达 850%~993%（软件渲染 llvmpipe 所致）。

## 涉及文件
- `/root/display.sh`：容器内图形显示管理脚本，三后端优先级 `external(平台WSLg) > windows(socat→VcXsrv) > local(容器内Xvfb+noVNC浏览器)`，可参数 `{start|stop|windows|local|status|check}`。
- `/root/.bashrc`（第 108~110 行）：`[ -f /root/display.sh ] && source /root/display.sh >/dev/null 2>&1`（由 docker/setup.sh 维护）。
- `~/catkin_ws`（git 仓库，origin=`git@github.com:Govis-Ink/aic26.git`）：分支替换/回退操作对象。

## 修改方案
### 一、Gazebo 显示问题（核心）
1. 诊断：`bash /root/display.sh status` 显示当前后端为 **local**（Xvfb 虚拟屏），`Windows X` 不可达（`host.docker.internal:6000 = 192.168.65.254` Connection refused）。
2. 根因：容器 `/tmp/.X11-unix` 实际是空 `tmpfs rw`（Docker Desktop 的 WSL 转发**没有把 WSLg 的 X0 socket 带进容器**），且 Windows 上无 VcXsrv，脚本只能退到 Xvfb 虚拟屏。
3. 修复（方案 B）：
   - `winget install marha.VcXsrv --silent` 安装 VcXsrv 21.1.16.1。
   - 启动 VcXsrv：`vcxsrv.exe :0 -ac -multiwindow -clipboard -wgl -nowgl`（**`-ac` 关闭访问控制是关键**，否则容器连接被拒）。
   - 容器内重测 `host.docker.internal:6000` 可达后，`bash /root/display.sh windows` 切换到 windows 后端（socat `UNIX-LISTEN:/tmp/.X11-unix/X0 → TCP4:192.168.65.254:6000`），Xvfb/fluxbox/x11vnc/noVNC 全部停止。
4. 结果：显示后端 = windows，`Windows X 可达`；重启 gazebo 后窗口直接弹到 Windows 桌面。

### 二、分支替换与回退（本次会话前半段，已完成）
- 目标：把 `feature/visual-slam`（注意真实分支名带 `feature/` 前缀）替换进 catkin_ws，重点 `aic26_perception` 包。
- 操作：先建备份分支 `backup/local-pre-visual-slam` 提交所有未提交改动；移走孤儿 submodule `src/gazebo_ros_model_color`（`.gitmodules` 已丢失但目录已初始化）；`git checkout -f -B feature/visual-slam FETCH_HEAD` 完成整仓切换。
- 回退：`git checkout -f main` 回到 `fbb90e9`，恢复 submodule 目录 + `git checkout backup/local-pre-visual-slam -- src/` 恢复实验改动。工作区状态与替换前一致。

## 结果与结论
- Gazebo 显示问题已定位为**显示后端退化到容器内 Xvfb 虚拟屏**，与代码分支无关；通过安装并启动 VcXsrv（`-ac`）+ `display.sh windows` 切到 socat 转发后端解决。
- 分支替换/回退均已安全完成，未提交改动由 `backup/local-pre-visual-slam` 分支与 `~/gazebo_ros_model_color_submodule_backup/` 目录保留。
- 注意点：VcXsrv 本次为手动启动，未设开机自启，Windows 重启后需重新启动；新开容器终端会因 `.bashrc` 自动 source display.sh 而自动复用 windows 后端。

## 变更记录
- 2026-10-02 11:21 首次整理
