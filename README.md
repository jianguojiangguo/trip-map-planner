# Trip Map Planner

**把零散的旅行输入，做成一份站得住的行程，和一页现场能用的地图。**

> 本仓库是 [hiyeshu/trip-map-builder](https://github.com/hiyeshu/trip-map-builder) 的改进版（MIT，保留原作者版权声明）。
> 上游给的是一条「规划 → 调研 → 出图」三阶段流水线；
> 本版在其之上补入 **区域资源普查 · 两轴咬合 · 出行前可行性校核 · 准入（票·约）台账**。
> 逐条差异与对比基准见 [UPDATES.md](UPDATES.md)。

## 它解决什么问题

旅行攻略的失败通常不在"选点"，而在"**串不起来**"——文字行程会骗人：
"上午 A、下午 B、晚上 C"读起来很顺，但它只证明了"我想去哪"，证明不了"我去得成"。

所以这份技能的核心不是"多列几个景点"，而是**把行程从愿望清单变成可校验的动线**。

## 核心模型：一张空间图 × 一个时间轴

| | 回答的问题 | 不成立会怎样 | 载体 |
|---|---|---|---|
| **空间图** | 点连得起来吗 | 以为顺路，实则折返 / 跨区 | 地图：片区 · 顺序 · 路径 |
| **时间轴** | 当天塞得下吗 | 排得满，走不完 | 日程：窗口 · 时长 · 缓冲 |
| **准入台账** | 进得去吗 | 到了门口进不了 | 票约页：已约 / 待抢 / 备选 |

**把两轴焊在一起的只有两件东西**：

1. **顺次编号** —— 把时间顺序投影到空间上，折返自己就现形；
2. **段间交通时长** —— 把空间距离换算成时间成本，当天塞不塞得下才算得出来。

没有这两件，图只是一堆孤立的点，日程只是一段看着通顺的叙述。

## 五段流程

```
① 定输入 → ② 摸家底 → ③ 编排（两轴）→ ④ 校核 → ⑤ 交付
```

| 段 | 一句话 | 产出 |
|---|---|---|
| ① 定输入 | 这次能用多少资源 | 约束清单（含主题偏好、节奏、体力、同行人） |
| ② 摸家底 | 区域有什么，摸成候选池 | 候选池表（带建议时长、开放预约、查证日期） |
| ③ 编排 | 两轴对齐 | 动线（空间）+ 时刻表（时间） |
| ④ 校核 | 三硬两软 | 可达 · 可容 · 可进入；合意 · 抗扰 |
| ⑤ 交付 | 三个载体同页咬合 | 地图页 · 日程 · 票约台账 |

**两条最容易被忽略、也最要紧的次序**：

- **偏好先定**，而且要当筛选条件用（主题类型是第一维度，决定选点方向）；
- **住宿后置**——先有景点路线，才有酒店综合决策，不是先订酒店再凑行程。

## 地图页能做什么（模板能力）

`assets/template.html` 是单文件模板，Leaflet 内联，**无需 API key、不依赖 CDN**：

- 交互地图 + 按天切换的时间轴卡片
- 每个点位三组直跳按钮：
  - **导航**：Apple Maps / Google Maps / 高德地图 App（scheme 直接唤起，不打开网页）
  - **小红书**：UA 检测，正常浏览器走 `xhsdiscover://`，微信/抖音等 WebView 内自动降级 m 站
  - **大众点评**：`food` / `drink` 类型自动启用，可用 `dianping: false` 关闭
- 预约按钮（可选）、支付方式标签
- 默认 Apple 设计系统，可整体换风格（见 [`references/map-build.md`](references/map-build.md) 第五节）

## 安装

```bash
git clone https://github.com/jianguojiangguo/trip-map-planner.git ~/.workbuddy/skills/trip-map-planner
```

技能目录随 agent 而定：Claude Code 用 `~/.claude/skills/`，Cursor 用 `~/.cursor/skills/`。

## 触发词

- "做个行程" / "行程规划" / "行程地图" / "旅游攻略"
- "plan my trip" / "trip map" / "build itinerary"
- "帮我查一下小红书上这家店怎么样"

## 共享记忆

技能会读取 `~/.trip-map-planner/MEMORY.md`，复用长期有效的旅行偏好：
主题偏好、节奏、餐饮、预算、支付与导航方式、历史输出。
它**不保存**原始截图、证件、订单号或完整聊天记录。

## 依赖

| 工具 | 用途 | 安装 |
|------|------|------|
| [OpenCLI](https://github.com/jackwener/OpenCLI) | 大众点评 adapter + 小红书调研 | `npm install -g @jackwener/opencli` |
| Chrome / Chromium | 浏览器 + 远程调试（真机自检） | — |
| [Leaflet.js](https://leafletjs.com) | 地图渲染（**内联进产物，不用 CDN**） | `assets/template.html` 内置 |
| [gh CLI](https://cli.github.com) | GitHub 仓库创建（可选） | 见官方文档 |

## 目录结构

```
trip-map-planner/
├── SKILL.md                    # 技能入口：两轴模型 + 五段流程 + 铁律 + 索引
├── README.md                   # 本文件
├── CLAUDE.md                   # 项目地图
├── UPDATES.md                  # 相对上游的改动清单
├── LICENSE                     # MIT
├── assets/
│   └── template.html           # 单文件地图模板（Leaflet 内联）
└── references/
    ├── CLAUDE.md               # references 局部地图
    ├── planning.md             # ①定输入 → ④校核：完整方法
    ├── map-build.md            # ⑤ 交付之地图：构建、自检、迁移
    ├── delivery.md             # ⑤ 交付之台账：票约台账与发布核查
    ├── research.md             # 调研：大众点评 / 小红书工作流
    └── interaction.md          # 对话：四拍格式与反模式
```

## 出处与许可

基于 **[hiyeshu/trip-map-builder](https://github.com/hiyeshu/trip-map-builder)** 扩展，许可 **MIT**，保留原作者版权声明（见 [LICENSE](LICENSE)）。

- 上游 demo：[tokyo-trip-pi.vercel.app](https://tokyo-trip-pi.vercel.app)（东京行程地图，源码 [hiyeshu/tokyo-trip](https://github.com/hiyeshu/tokyo-trip)）
- 上游仓库内**没有 LICENSE 文件**（仅在 `README.md` 末尾以一行 `MIT` 声明许可）。本仓库补上一份 MIT 全文，把原作者版权声明放在首位——这不是新增限制，而是把作者已声明的许可落成可随副本传递的正式文本。
- 底图瓦片来自第三方地图服务，**其服务条款与使用范围请自行确认**，本项目不代为授权。
