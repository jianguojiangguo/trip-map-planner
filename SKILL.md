---
name: trip-map-planner
description: >
  旅行行程策划 + 交互式行程地图生成。从零散输入（截图 / 清单 / 已订票）出发，
  先把区域资源普查成候选池，再按「一张空间图 × 一个时间轴」编排，
  校核可达 / 可容 / 可进入，最后产出一页手机可用的单文件地图
  （含日程、票约台账、一键导航）。
  End-to-end trip planning and single-file itinerary map. Pipeline:
  resource survey → two-axis scheduling (space × time) → feasibility check →
  mobile map page. Use when the user asks to plan a trip, build an itinerary,
  make a trip map, or says 行程规划 / 行程地图 / 旅游攻略 / 做份攻略 /
  生成行程 / 帮我排行程 / plan my trip / trip map. Covers scattered inputs to a
  deployable reference map with navigation links, reservation tracking,
  and 大众点评 / 小红书 signals.
---

# 行程策划与地图

把零散的旅行输入，做成一份**站得住的行程**，和一页**现场能用的地图**。

产出的是一份**参考坐标**，不是旅途中必须执行的脚本——天气、当前位置、体力、饥饿都可以覆盖它。
行程的价值在于：今天在哪个片区、哪些是锚点、哪些可以放弃、饭去哪儿找。

---

## 一、模型

### 1.1 一份行程靠什么成立

一份行程 = **一张空间图 + 一个时间轴**，外带一道准入门槛。三者同时成立，行程才成立。

| | 回答的问题 | 不成立会怎样 | 载体 |
|---|---|---|---|
| **空间图** | 点连得起来吗 | 以为顺路，实则折返 / 跨区 | 地图：片区 · 顺序 · 路径 |
| **时间轴** | 当天塞得下吗 | 排得满，走不完 | 日程：窗口 · 时长 · 缓冲 |
| **准入台账** | 进得去吗 | 到了门口进不了 | 票约页：已约 / 待抢 / 备选 |

**把两轴焊在一起的只有两件东西：**

1. **顺次编号** —— 把时间顺序投影到空间上，折返自己就现形；
2. **段间交通时长** —— 把空间距离换算成时间成本，当天塞不塞得下才算得出来。

没有这两件，图只是一堆孤立的点，日程只是一段看着通顺的叙述。

### 1.2 两轴正交，不能互替

点位由两个坐标共同定位，**它们回答的是两个不同的问题**：

| 轴 | 回答 | 决定什么 | 落到产品上 |
|---|---|---|---|
| **主题轴**（游什么） | 这个点讲什么内容 | 检索清单 · 选点方向 | 卡片标签 / 筛选器 / 配色 |
| **功能轴**（干什么） | 这个点在行程里什么角色 | 排布配比 · **地图标记画什么** | 图标的形状与颜色 |

- **主题轴 12 项**（选 1–2 个作主轴，其余为副轴；选不出主轴用「混合」兜底）：
  `历史人文 · 文学艺术 · 自然山水 · 市井民俗 · 美食风物 · 建筑园林 · 工业遗产 · 红色纪念 · 亲子研学 · 户外运动 · 演出体验 · 混合`
- **功能轴 6 角色**：`核心目标 · 顺路穿插 · 餐饮 · 休憩补给 · 住宿 · 交通枢纽`

> **常见错误**：把两者混成一张清单（如"景点 / 餐饮 / 文学地标 / 民俗"）。它们永远互斥不了——文学地标本身就是景点，非遗餐厅既是民俗又是餐饮。
> **正确用法**：**标记画功能**（它在行程里什么角色），**主题交给标签与筛选器**（它讲什么内容）——地图图标因此有了判据：**画功能，不画主题**。

---

## 二、五段流程

```
① 定输入 → ② 摸家底 → ③ 编排（两轴）→ ④ 校核 → ⑤ 交付
```

| 段 | 一句话 | 产出 | 详读 |
|---|---|---|---|
| ① 定输入 | 这次能用多少资源 | 约束清单 | `references/planning.md` |
| ② 摸家底 | 区域有什么，摸成候选池 | 候选池表 | 同上 |
| ③ 编排 | 两轴对齐，成动线 + 时刻表 | 行程方案 | 同上 |
| ④ 校核 | 三硬两软，过不了就改 | 验收结论 | 同上 |
| ⑤ 交付 | 三个载体，同页咬合 | 地图页 · 日程 · 台账 | `references/map-build.md` · `references/delivery.md` |

### ① 定输入 · 约束边界

定下"这次能用多少资源"，后面每一步都挂在它上面。

- **锚点**：日期与天数、到达 / 返程时刻与航站楼
- **偏好**：主题类型（第一维度）· 节奏 · 体力 · 预算 · 忌口
- **同行人**：谁、几岁、能否分开行动（准入要同步、节奏取交集）
- **已排除项**：明确不去的，免得反复推荐

两条顺序铁律：**偏好先定**（且当筛选条件用，不是读一遍）；**住宿后置**（先有景点路线，才有酒店）。

### ② 摸家底 · 候选池

起点是**区域资源**，不是用户的清单，更不是模型的记忆。

不普查，选点就是记忆驱动 → 必然漏项 → **用户点名要去的地方没排进去**。

- 检索维度由**本次主题偏好**决定（这次主题是"文学 + 市井"，就按这两个主题捞，不是固定四类）
- 逐级下钻行政区划（市 → 区县 → 镇 / 街区），保证不漏县域
- 候选池每个点必带：**建议游览时长 · 开放时间与闭馆日 · 是否预约限额 · 来源与查证日期**

### ③ 编排 · 两轴对齐（核心）

- **空间轴**：片区聚类 → 排顺序 → 定交通方式与路径
- **时间轴**：开放窗口 → 游玩时长 → 段间交通 → 用餐 → 缓冲

两条轴同时满足，才叫排完。**每个点必须有两个数**：玩多久、到下一个点怎么走多久——缺了这两个，当天是否成立根本无法验证。

### ④ 校核 · 三硬两软

- **三硬**（任一不过，那天就不成立 → 只能删点 / 换天 / 换锚点）：**可达** · **可容** · **可进入**
- **两软**（决定值不值）：**合意**（主题匹配 · 冷热搭配 · 餐饮落位）· **抗扰**（室内备选 · 信息时效）

### ⑤ 交付 · 三个载体

| 载体 | 承载 | 最低要求 |
|---|---|---|
| **地图** | 空间轴 | 点位 + 动线 + 编号 + 段间时长；手机可开、断网可用、一键导航 |
| **日程** | 时间轴 | 按天分组的时刻表 |
| **票约台账** | 准入 | 已约 / 待抢 / 抢不到的备选 + 时点提醒 |

> 票约台账不是"提醒"，是**行程的准入台账**：有 deadline、有名额稀缺性、抢不到要改行程。它与两张图并列，是第三个工具。

分享前打码住宿与个人信息。

---

## 三、铁律

### 业务判断

1. **偏好先定，且当筛选条件用**。主题决定选点方向，节奏与体力决定点数上限。
2. **先普查、再选点**。没摸过的资源不算数；候选池必须带查证日期。
3. **住宿后置于片区**。正确顺序：主题偏好 → 选点 → 片区与动线 → **住宿综合决策**（位置 · 价位 · 通勤 · 停车一起权衡）。
   例外：住宿已订且不可改时，它是**既定约束**，允许回头影响片区划分——但主导权仍在景点路线。
4. **一天一个主区域**；一天最多一个重预约点；第一天下轻手；最后一天不跑远。
5. **替用户删东西，并说明删了什么、为什么**。行程不是越满越好，是越顺越好。
6. **每个点两个数**：建议游玩时长 + 到相邻点的交通方式与时长。
7. **点名要去的核心地标必须单独成节点并给明确时段**，别只排成"顺路经过 / 在附近吃饭"；优先排在它人最少的时段。
8. **预约与限流信息每次实查并标注查证日期**（政策会过期）；注意"周一闭馆 × 法定节假日"这类组合。
9. **餐厅是补给点，不是锚点**：先保当天片区顺路，再给 2–3 个候选；只有预约餐、强目的餐、用户指定店才允许反向影响路线。排队时长要算进当天时间预算。

### 坐标与路径

10. **坐标必须转换**：输入（高德 / 腾讯）是 GCJ-02，OSRM 路网是 WGS84，中国境内差 300–500m。
    链路 = 输入 GCJ-02 → `gcj2wgs()` → OSRM → 返回 WGS84 → `wgs2gcj()` → 画在高德瓦片上。
11. **OSRM 必须 `overview=full`**，再按 8m 阈值抽稀；`simplified` 会把路线画成斜切街区的折线。
12. **卡片显示名 ≠ 路径 key**：每个点带 `key` 字段，构建脚本自检 `MISSING_ROUTES`（必须为 none）。
13. 拿不到可靠地理编码时，页面**明确标注"近似定位，导航按名称搜索"**。

### 图面

14. **先算对比度，再改颜色**：荧光色（亮蓝 / 亮绿 / 黄）写白底一律过不了 4.5:1。荧光色只能写在深底上。
15. **地图瓦片绝不加 `sepia` / 大幅降饱和**——染色会连路线标识一起毁掉。
16. 浅色小图形要同时满足 **深描边 + 投影 + ≥40px**，三条缺一就"隐身"。
17. **标记画功能**（主目标 / 顺路 / 吃 / 住 / 枢纽），主题交给标签与筛选器。
18. **地图库内联，零 CDN**（现场网络差会白屏）；模板取上游 main 最新版。
19. 地图标记**不做入场动画**（会卡在首帧）；若做了沿路装饰，装饰 marker 也要进 `markers` 数组，否则切天翻倍。

### 交付

20. **真机自检必须真点一次交互**（tab / 票约 / 全屏），光读源码发现不了"路由缺分支"这类 bug。
21. 手机宽度下量 `getBoundingClientRect`，确认图例没压住 Leaflet 版权行。
22. 分享前打码住宿与个人信息；公开发布前过四关（许可形态 / 逐文件比对上游 / 数字可复算 / 隐私扫描）。

---

## 四、文件地图

| 文件 | 什么时候读 |
|---|---|
| `references/planning.md` | 排行程：①定输入 → ④校核 的完整方法、输入输出模板、主题词表、常见坑 |
| `references/map-build.md` | 出图：坐标与路径、标记系统、皮肤、真机自检、迁移第二条线路；附录含改文件手法（沿路装饰为可选，见第六节） |
| `references/delivery.md` | 交付：票约台账的建法与发布前核查 |
| `references/research.md` | 调研：大众点评 / 小红书工作流、平台可用性、信号怎么读 |
| `references/interaction.md` | 与用户对话：四拍格式（Re-ground → Simplify → Recommend → Options）与反模式 |
| `assets/template.html` | 生成地图页的底版 |

---

## 五、Shared memory

Before planning or building, read `~/.trip-map-planner/MEMORY.md` if it
exists. Use it only for durable traveler context:

- **主题偏好**（历史人文 / 文学艺术 / 自然山水 / 市井民俗 / …）
- pace preference
- food and drink preferences
- budget habits
- payment and navigation preferences
- previously generated trip outputs
- recurring constraints and unresolved follow-ups

If the file does not exist, continue normally. Do not block on memory setup.

Do not store raw screenshots, passport data, booking codes, full chat logs, or
other sensitive/private material.

After each completed trip plan, research pass, or map build, update
`~/.trip-map-planner/MEMORY.md` with only durable facts:

```md
# Trip Map Builder Memory

## Traveler Defaults
- Departure city:
- 主题偏好 (theme):
- Pace:
- Food preferences:
- Budget habits:
- Payment preference:
- Navigation preference:
- Language preference:

## Past Trips
| Trip | Dates | Destination | Output | Notes |
|------|-------|-------------|--------|-------|

## Reusable Preferences
-

## Open Threads
-
```

---

## 六、依赖与出处

| 工具 | 用途 | 安装 |
|------|------|------|
| [Leaflet.js](https://leafletjs.com) | 地图渲染（**内联进产物，不用 CDN**） | `assets/template.html` 内置 |
| [OSRM](http://project-osrm.org/) | 真实路网路径（`overview=full`） | 公共 API |
| [OpenCLI](https://github.com/jackwener/OpenCLI) | 大众点评 adapter + 小红书调研 | `npm install -g @jackwener/opencli` |
| Chrome / Chromium | 浏览器 + 远程调试（真机自检） | 已有 |
| [gh CLI](https://cli.github.com) | GitHub 仓库创建（可选） | `brew install gh` |

**出处**：本技能基于 [hiyeshu/trip-map-builder](https://github.com/hiyeshu/trip-map-builder)（MIT）扩展。
上游提供三阶段流水线（规划 → 调研 → 出图）与地图模板；本版在其之上补入**区域资源普查、两轴咬合、可行性校核、准入台账**四块。
许可与改动清单见 `references/delivery.md` 末节。
