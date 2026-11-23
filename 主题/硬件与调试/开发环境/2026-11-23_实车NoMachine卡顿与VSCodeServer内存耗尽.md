---
title: 实车 NoMachine 卡顿：VS Code Server 内存耗尽与残留脚本
date: 2026-11-23
updated: 2026-11-23
project: 开发环境
type: 报错排查
themes: [环境与工具链, 硬件与调试]
tags: [真车]
status: 已解决
---

# 实车 NoMachine 卡顿：VS Code Server 内存耗尽与残留脚本

> 整理日期：2026年11月23日 19:17
> 项目：开发环境 ｜ 类型：报错排查 ｜ 状态：已解决 ｜ 主题：`环境与工具链`、`硬件与调试` ｜ 标签：`真车`

## 背景 / 目标

在使用 NoMachine 远程操作实车主板时，桌面明显卡顿，怀疑车端堆积了僵尸进程或 ROS 残留。目标是确认 CPU / 内存 / 进程的真实占用，找出可安全清理的部分。

## 环境信息

| 项 | 值 |
|---|---|
| 主机 | `mowen-JNE10`，Intel N100 4 核 / 7.5 GB RAM，Ubuntu 20.04 |
| 远程桌面 | NoMachine（`DISPLAY=:1001`） |
| 开发工具 | VS Code Remote-SSH（服务端）+ 车端本地 VS Code GUI + Pylance + C/C++ 扩展 |
| 其他 | station 上位机 `server.py`；当时 ROS bringup 已停止 |

## 现象与关键数据

卡顿最严重时（17:28 左右）采集：

| 指标 | 数值 |
|---|---|
| load average | **30.44 / 20.59 / 12.36**（4 核） |
| 内存 | 7.5 GiB 总，已用 **7.0 GiB**，空闲 128 MiB，可用 138 MiB |
| Swap | **2.0 GiB 全部用满** |
| 进程总数 | 330 |
| 僵尸进程 | **仅 2 个**（`[pet]`、`[xdg-user-dir]`） |

随后负载继续上升到 47.12，SSH 出现 `Connection timed out during banner exchange`，station 的 `8780` 也超时，但 `ping 192.168.1.15` 仍正常（6–187 ms），说明不是网络断开，而是主机资源耗尽、sshd/HTTP 服务拿不到 CPU。

## 根因

### 1. 不是僵尸进程

只有 2 个 `defunct`，由父进程稍后回收，与卡顿无关。

### 2. 主要占用

| 进程 | 占用 |
|---|---|
| `/tmp/gauge_final_check.py` | **单核 90% CPU**，从 VS Code 集成终端启动，已跑 2 分半 |
| VS Code Server extensionHost ×2 | 各约 17% CPU；内存 15.6% + 13.5% |
| Pylance ×2 + cpptools ×3 | 再占约 2 GB |
| 车端本地 VS Code GUI（renderer / utility） | 18.6% + 13.8% CPU，utility 占 26% 内存 |
| station `server.py` | 5.8% CPU，不是主因 |

`ps` 检查确认：**没有** `roslaunch`、`rosmaster`、`gzserver`、`_detector.py`、`camera_node.py`、`mission_controller.py` 等 ROS 残留。

### 3. 触发链路

VS Code Remote + 本地 VS Code GUI + 两套 Pylance/cpptools 已把 7.5 GB 内存耗尽；再叠加一个 90% CPU 的 `/tmp/gauge_final_check.py`，swap 被完全打满，进入持续换页，最终连 SSH banner 都发不出来。

## 处理过程

1. 杀掉残留脚本：

```bash
kill 148960          # python3 -u /tmp/gauge_final_check.py
```

   `148960 gone`，`pgrep -af "/tmp/.*\.py"` 为空。

2. 杀脚本后 load 从 47.12 降到 24.16，可用内存从 127 MiB 回到 986 MiB；但 VS Code 栈仍占 3–4 GB，swap 仍满。

3. 随后主机重启（uptime 归零），状态恢复：

```text
load average: 0.23, 0.05, 0.02
内存：已用 366 MiB，可用 6.9 GiB
Swap：2.0 GiB，已用 0
```

本次没有执行 `pkill -f vscode-server`，重启完成了清理。

## 减少复发的建议

- 只开**一个** VS Code Remote 窗口；两个窗口会起两套 extensionHost/Pylance。
- 大工作区建议关闭 Pylance 全量索引：

```json
"python.analysis.indexing": false,
"python.analysis.userFileIndexingLimit": -1
```

- 不做 C++ 开发时禁用 C/C++ 扩展（当时有 3 个 `cpptools` 进程）。
- 不要在主板上同时跑 VS Code 索引、Gazebo/检测脚本和整车 bringup；N100 只有 4 核 / 7.5 GB。
- 高负载时 SSH 仍可用简化握手重试（实测）：

```bash
ssh -o ConnectTimeout=60 -o BatchMode=yes -c aes128-ctr mowen@192.168.1.15
```

  如果连 banner 都超时、但 ping 正常，优先怀疑内存/swap 抖动，而不是网络。

## 结果与结论

- 实车卡顿的根因是 **VS Code Server 栈内存耗尽 + 残留仪表测试脚本吃满一个核**，不是僵尸进程、也不是 ROS 漏进程。
- 杀掉脚本后负载与内存开始恢复；重启后回到空闲状态（load 0.2、可用内存 6.9 GB）。
- 后续按“单窗口、关索引、不同时跑重负载”的规范使用，可避免复发。

## 相关记录

- [Gazebo 仿真界面卡顿定位与降载优化](../项目/ROS仿真车/2026-10-25_Gazebo仿真卡顿优化.md) —— 同一块 N100 板的资源超订分析，已记录 VS Code Server + NoMachine 的 CPU 占用
- [AIC26 实车相机与视觉检测稳定性排障](../项目/ROS仿真车/2026-11-15_AIC26实车相机与视觉检测稳定性排障.md)
- [A/B 街社区人员与 ebike_park 分区统计](../项目/ROS仿真车/2026-11-23_AB街人员与ebike_park分区统计.md)

## 变更记录

- 2026-11-23 19:17 首次整理
