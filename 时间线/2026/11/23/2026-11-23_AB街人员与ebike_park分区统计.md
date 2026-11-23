---
title: A/B 街社区人员与 ebike_park 分区统计
date: 2026-11-23
updated: 2026-11-23
project: ROS仿真车
type: 功能开发
themes: [感知与视觉]
tags: [ros, perception, 真车]
status: 部分解决
---

# A/B 街社区人员与 ebike_park 分区统计

> 整理日期：2026年11月23日 19:17
> 项目：ROS仿真车 ｜ 类型：功能开发 ｜ 状态：部分解决 ｜ 主题：`感知与视觉` ｜ 标签：`ros`、`perception`、`真车`

## 背景 / 目标

实车 `real_nostrafe_tasks.yaml` 新增了 6 个人员点位：

- A 街：`people_a1`(station_1)、`people_a2`(station_2)、`people_a3`(station_3，分支内)
- B 街：`people_b1`(station_3 另一侧)、`people_b2`(traffic_light_2)、`people_b3`(station_4_to_n / station_4_to_s，分支内)

每个点位 `targets: ["people", "ebike_park"]`。需求是：

1. A/B 街分别统计社区人员（数字类别 `1..16`）与非社区人员（`a`/`b`）；
2. **不做跨点位去重**，各点位直接相加；
3. 同时统计 `ebike_park`（违停电动车）数量；
4. **不新增语音播报**；
5. 六个点位全部结束后，输出一次：

```text
社区内共有 xx 人，其中 A 街人数 xx 人，B 街人数 xx 人，
发现 xx 名非社区人员在 A 街，xx 名非社区人员在 B 街。
```

其中“A/B 街人数”= 该区域社区人员 + 非社区人员之和。

## 环境信息

| 项 | 值 |
|---|---|
| 实车 | `mowen@192.168.1.15`，N100 4 核 / 7.5 GB RAM |
| 工作区 | `~/aic26_ws_112`，`backend:=real` 默认 `config_prefix=real_nostrafe` |
| 任务/航点 | `aic26_decision/config/real_nostrafe_tasks.yaml`、`real_nostrafe_waypoints.yaml` |
| people 检测 | `person.onnx` 18 类：`1..16` 社区、`a`/`b` 非社区；发布 `total/community/intruder/intruders/boxes` |
| ebike_park 检测 | 复用 `layout_detector.py`，`ebike_park.onnx`，类别 `ebike`，conf 0.3，发布 `total/present/max_conf/boxes` |
| 结果目录 | `~aic26_ws_112/src/aic26_perception/results`（launch 的 `results_dir`） |

## 现状与缺口

- 检测层已经能给出社区/非社区人数和违停车辆数，但决策层 `mission_controller` 只把结果写进 `self.results[target]`，每过一次点位就被覆盖；`results/people.json`、`summary.json` 也只保留最后一次。
- `summarize` 任务只播固定语句和 ebike 去重，不含人员分区统计。
- `build_message()` 对 `ebike_park` 固定返回空串，逐点数量无人保存。
- 实车工作区此前已有一版“按街道合并”的代码：`street_map` 只认旧任务名 `people_a/b/c/d/e/f`，用 `set` 做跨点位**去重**，并在 `summarize` 语音播报。这与新任务名、新口径（不去重、不播报）都不兼容。

## 涉及文件

| 文件 | 改动 |
|---|---|
| `aic26_decision/scripts/mission_controller.py` | 点位记录、A/B 求和、汇总句、`zone_summary.json`、ResultViewer 分区段 |
| `aic26_decision/config/real_nostrafe_tasks.yaml` | 本次不改；6 个点位及其 `targets` 已存在 |
| `aic26_perception/scripts/people_detector.py`、`layout_detector.py` | 本次不改，直接消费现有字段 |

备份：`mission_controller.py.bak_20261123_zone_stats`（改动前的实车版本）。

## 修改方案

### 1. `MissionController` 新增点位统计

`__init__` 新增：

```python
self.street_map = {
    'people_a1': 'A 街', 'people_a2': 'A 街', 'people_a3': 'A 街',
    'people_b1': 'B 街', 'people_b2': 'B 街', 'people_b3': 'B 街',
    # 兼容旧任务名
    'people_a': 'A 街', 'people_b': 'A 街',
    'people_c': 'B 街', 'people_d': 'B 街',
    'people_e': 'B 街', 'people_f': 'A 街',
}
self.people_streets = {}   # street -> {'community': int, 'intruder': int}
self.ebike_illegal = {}    # street -> 违停总数（不去重）
self.results_dir = rospy.get_param('~results_dir', default_results_dir())
self.zone_stats = {}       # task name -> 每点记录
self.zone_lock = threading.Lock()
self.people_sentence = ''
self.people_sentence_emitted = False
```

每个点位记录结构：

```python
{
  'task': 'people_a1', 'waypoint': 'station_1', 'group': 'A',
  'people': {'community': 2, 'intruder': 0, 'total': 2,
             'intruders': [], 'seen_classes': ['1', '7']},
  'ebike_park': {'total': 1, 'present': True, 'max_conf': 0.6},
}
```

`_record_zone_stop(task, results)` 在 inspect 的检测循环结束后调用一次；即使转向失败也调用（结果记 `None`），保证六点齐全判断不被卡住。结果缺失不按 0 计。

### 2. A/B 求和与汇总句

- `people_streets` 按街累加 `community` 与 `intruder`，**不去重**；`street_total = community + intruder`。
- `ebike_illegal` 按街累加 `ebike_park.total`，原来的“取最大值”改为累加。
- 当 `people_a1/a2/a3/b1/b2/b3` 六个 key 全部出现在 `zone_stats` 后，`_maybe_emit_people_sentence()` 生成并输出一次：

```text
社区内共有 18 人，其中 A 街人数 8 人，B 街人数 10 人，
发现 2 名非社区人员在 A 街，0 名非社区人员在 B 街。
```

缺失点位会排除在求和之外，并在句末追加 `（缺少结果：people_b2）`，同时 `logwarn` 列出缺失名单。

### 3. 落盘与展示

- 每记录一个点位就原子写 `<results_dir>/zone_summary.json`（temp + `os.replace`），包含：
  - `sentence`
  - `groups.A/B/other`（community、intruder、street_total、ebike_park、stops、missing）
  - `stops` 每点明细
  - `total`（A+B 合计）
- `ResultViewer` 新增 `zone_summary` 回调，Tk 面板末尾显示 `== people zones (no dedup) ==`，`summary.json` 增加 `zones` 字段。
- `_log_zone_summary()` 在任务结束时兜底打印 A/B/other 汇总；若六点未齐，只 `logwarn`，不输出汇总句。
- 全程不调用 `speak()`；`summarize` 中原有的 people 语音汇总已移除。`louyuD`/`meter` 的 people 若不参与 A/B，只进 `other`，不混入汇总句。

## 验证

### 静态与单元测试

- `python3 -m py_compile mission_controller.py` 通过；
- `catkin_make --pkg aic26_decision` 返回 `BUILD_RC=0`；
- 三组远端单元测试通过：
  1. 六点求和与句子格式（构造 16 人场景，A 街 7 / B 街 9 / 非社区各 2）；
  2. `people_b2` 无结果时不计入、句子带“缺少结果”提示；
  3. `louyuD` 只进 `other`，不进汇总句；
- 用真实任务文件模拟两条分支：blocked 在 `people_b3` 后输出，passable 在 `people_b2` 后输出。

### 实车运行日志

2026-11-23 17:11–17:15 的实车运行（blocked 分支）实际输出了：

```text
zone people_a1: people community=2 intruder=0 total=2 | ebike_park=1
zone people_a2: people community=2 intruder=1 total=3 | ebike_park=0
zone people_a3: people community=2 intruder=1 total=3 | ebike_park=0
zone people_b1: people community=4 intruder=0 total=4 | ebike_park=1
zone people_b2: people community=3 intruder=0 total=3 | ebike_park=1
zone people_b3: people community=3 intruder=0 total=3 | ebike_park=1
PEOPLE GROUP SUMMARY: 社区内共有 18 人，其中 A 街人数 8 人，B 街人数 10 人，发现 2 名非社区人员在 A 街，0 名非社区人员在 B 街。
```

与手算一致：A 街 community `2+2+2=6`、intruder `0+1+1=2`、合计 8；B 街 community `4+3+3=10`、intruder 0、合计 10；总计 18。

同一次运行后段在 `rubbish`/`sort` 之后反复刷 `detector protocol v2` 并出现 `no complete car/rubbish result`，任务没有完整走完；这是检测器/负载层面的问题，不影响本次汇总句已正确输出。

## 实车终端输出清单

实车 `bringup.launch backend:=real` 下所有 `output="screen"` 节点都会往终端打日志。与本次功能相关的关键行：

| 类别 | 终端示例 |
|---|---|
| 任务流 | `task: people_a1`、`goal sent: station_1 (-1.00, 1.70, 3.14)` |
| 检测触发 | `detect start: people (55825a6a:2)`、`detect start: ebike_park (...)` |
| 检测器 | `people request started/finished (session, id, 2 frames, counts=[...], mode=..., selected=...)`；`ebike_park request started/finished (...)` |
| **分区统计** | `zone people_a1: people community=2 intruder=0 total=2 \| ebike_park=1` |
| **汇总句** | `PEOPLE GROUP SUMMARY: 社区内共有 … 人，其中 A 街人数 … 人，B 街人数 … 人，发现 … 名非社区人员在 A 街，… 名非社区人员在 B 街。` |
| 其他播报 | `speak: <文本>`（所有送入 `/speech` 的内容都会先打这一行） |
| 楼宇火情 | `speak: 星河大厦发现火灾隐患 3 个` |
| 车辆 | `speak: 检测到车辆，但未能识别车牌。` / `1号停车场车牌号为…` |
| 仪表 | `gauge_reader: camera requested 1920x1080 @ 10 fps`、`gauge_reader: FALLBACK to the camera's actual size`、`speak: 温度表读数为 42度` |
| ebike | `ebike dedup result: status=... total=... views=2 usable=True`；summarize 时 `speak: 综合两次观察，…` |
| 垃圾桶 | 每个桶一行 `speak: 可回收垃圾桶状态为打开，垃圾桶里的垃圾为塑料瓶，投放正确。` |
| 异常 | `no complete … result within …s`、`detector protocol v2` 反复、`gauge_reader: ignoring stale stop` |

注意：people / ebike_park 任务上的 `message: "Zone A ..."` 不会出现——代码对这两个 target 直接 `continue`，逐点只输出 `zone` 行；人员汇总句也只进日志/Tk/JSON，不进语音。

## 结果与结论

- 六点分区统计、A/B 汇总句、`zone_summary.json`、Tk 分区段均已实现，并在实车真实运行中输出了正确结果。
- 不新增语音；people / ebike_park 逐点不播报。
- `ebike_park` 口径从“每街取最大值”改为“逐点累加”，与“不考虑去重”一致；旧 `people_*` 任务名保留映射。
- **待验证/遗留**：
  1. 缺失结果路径只在单元测试中验证，尚未在实车跑到某个点位超时的场景；
  2. 同一次实车任务后段因检测器协议等待没有跑完，需要单独排查（与本次功能无关）；
  3. `sim_nostrafe_tasks.yaml` 仍是旧任务名，因此仿真默认配置不会触发汇总句，只有实车 `real_nostrafe` 会；
  4. `people_int8.onnx` 的类名是 `p1..p16/pF1/pF2`，与脚本写死的 `1..16/a/b` 不匹配，切模型前需确认 launch 加载的是 `person.onnx`。

## 相关记录

- [AIC26 实车相机与视觉检测稳定性排障](2026-11-15_AIC26实车相机与视觉检测稳定性排障.md)
- [ebike 两站视觉去重与独立去重节点](2026-11-16_ebike两站视觉去重与独立节点.md)
- [DELI 9011 温湿度表盘双指针读数方案](2026-11-15_DELI9011温湿度表盘读数方案.md)
- [AIC26 垃圾分类 sort 模型替换与 rubbish bin 自动检测接入](2026-11-15_AIC26垃圾分类sort检测接入与置信度调优.md)
- [实车 NoMachine 卡顿：VS Code Server 内存耗尽与残留脚本](../开发环境/2026-11-23_实车NoMachine卡顿与VSCodeServer内存耗尽.md)

## 变更记录

- 2026-11-23 19:17 首次整理
