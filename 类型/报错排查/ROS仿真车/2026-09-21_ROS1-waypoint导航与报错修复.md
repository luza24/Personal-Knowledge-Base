---
title: ROS1 智慧社区 waypoint 导航实现与报错修复
date: 2026-09-21
updated: 2026-09-21
project: ROS仿真车
type: 报错排查
themes: [导航与运动控制]
tags: [ros, move_base]
status: 部分解决
---

# ROS1 智慧社区 waypoint 导航实现与报错修复

> 整理日期：2026年9月21日 22:48

## 环境信息
- 操作系统 / 运行环境：Windows 11 + Docker Desktop 容器（`root@docker-desktop`）
- ROS 版本：ROS1（Noetic，Ubuntu 20.04 基础，Python3）
- 功能包：`aic26_navigation`（`~/catkin_ws/src/aic26_navigation/`）
- 建图：gmapping；定位：AMCL；导航：move_base + DWA 局部规划器
- 地图文件：`maps/nav.pgm` + `maps/nav.yaml`

## 报错记录

1. **`rosrun ... waypoint_nav.py` → `/usr/bin/env: 'python': No such file or directory`**
   - 原因：环境只有 python3，脚本 shebang 写了 `python`。

2. **`IndentationError: unexpected indent`（第 3 行 `import rospy`）**
   - 原因：整个脚本被多缩进了 2 个空格（复制粘贴带入）。

3. **`catkin_make` 失败**
   - 原因：`CMakeLists.txt` 的 `find_package(catkin REQUIRED COMPONENTS ...)` 里写了 `math`。`math` 是 Python 标准库，不是 ROS 包。

4. **RViz 中小车显示在地图中间，但 Gazebo 里车在地图右上角**
   - 原因：AMCL 未设初始位姿，默认认为车在 map 原点 (0,0)，而建图出发点才是 map 原点。

5. **RViz 地图围绕小车旋转 180° 才对得上**
   - 原因：AMCL 初始位姿 x、y 对，但朝向 yaw 反了 180°。

6. **waypoint 失败 `state=1`（ACTIVE），每个失败点卡 20 秒**
   - 原因：车道离墙 0.32m 太挤。`inflation_radius=0.3` + footprint 半宽 ~0.15 ≈ 0.45m，车开不进 0.32m 的车道，西侧墙角需原地转 90° 卡死。

7. **到达目标点后停得久**
   - 原因：① 失败点干等 20 秒超时；② `yaw_goal_tolerance=0.4` 导致到位后原地转朝向；③ 速度慢（max_vel_x=0.3）。

## 涉及文件

- `scripts/waypoint_nav.py` — waypoint 多点导航脚本（发 MoveBaseGoal）
- `config/costmap_common_params.yaml` — footprint、inflation_radius
- `config/dwa_local_planner_params.yaml` — 速度、加速度、到达容差
- `config/global_costmap_params.yaml` / `local_costmap_params.yaml` — 代价地图
- `launch/nav.launch` — map_server + amcl + move_base + rviz
- `CMakeLists.txt` — find_package 依赖
- `maps/nav.pgm` + `nav.yaml` — gmapping 建图结果
- Gazebo world 文件 — 场地：gallery_floor(11×8)、map_walls 四面墙、traffic_light×2、cp 车牌模型×3

## 修改方案

- **报错 1**：shebang `#!/usr/bin/env python` → `#!/usr/bin/env python3`。
- **报错 2**：脚本顶格写，`import` 前不能有空格。
- **报错 3**：`CMakeLists.txt` 的 find_package 中删除 `math`。
- **报错 4**：用 RViz `2D Pose Estimate` 手动设初始位姿，或 `nav.launch` 写死 `initial_pose_x/y/a`。
- **报错 5**：`initial_pose_a` 加/减 π（3.14159）。
- **报错 6/7（方案 A）**：车道由离墙 0.32m 外移到 0.5m；东侧保持 0.92m（x=1.13，因有 cp 车牌模型需避开）；`inflation_radius` 0.3→0.25。
- **提速 + 少磨蹭**：
  - `max_vel_x` 0.3→0.5、`max_vel_theta` 1.0→1.5、`acc_lim_x` 0.5→1.0；
  - `yaw_goal_tolerance` 0.4→6.28（2π，朝向无所谓，到位即成功，不再原地转）；
  - 脚本超时 `wait_for_result(20)` → 8 秒 + 超时后 `cancel_goal()`；
  - `rospy.sleep(0.5)` → 0.05。

### 关键坐标（方案 A）

- 墙内侧面：north y=2.05 / south y=-2.05 / west x=-2.05 / east x=2.05（墙厚 0.1，中心 ±2.1）
- 车道线：北 y=1.55、西 x=-1.55、南 y=-1.55、东 x=1.13
- 闭合环线 9 点：`(1.13,1.55) → (0,1.55) → (-1.55,1.55) → (-1.55,0) → (-1.55,-1.55) → (0,-1.55) → (1.13,-1.55) → (1.13,0) → (1.13,1.55)`

### 导航启动顺序

```
catkin_make → source devel/setup.bash
roslaunch aic26_navigation gazebo.launch   # 车 + 场景
roslaunch aic26_navigation nav.launch      # map_server + amcl + move_base
# RViz 点 2D Pose Estimate 定初始位姿，激光贴墙
rosrun aic26_navigation waypoint_nav.py    # 发路径点
```

## 结果与结论

- 导航前半段（北侧 y=1.73 直线）已跑通，`reached!`。
- 西侧墙角点失败（state=1）确认是车道离墙太近所致，已采用方案 A（0.5m 车道 + 降膨胀 + 提速 + 放宽朝向容差）给出完整改后文件。
- 后续参数调优中，待实跑验证。

## 变更记录
- 2026-09-21 22:48 首次整理
