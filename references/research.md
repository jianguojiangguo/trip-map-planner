# 平台调研工作流

调研服务于 ② 摸家底（候选池）与 ④ 校核（合意）。

**分工先分清**：

| 平台 | 给什么信号 |
|---|---|
| **大众点评** | **硬信号**：口味、排队、踩雷、价格、区域、值不值得 |
| **小红书** | **软信号**：氛围、近期体验、拍照、软性提醒 |

不要拿社媒种草当硬口碑，也不要拿点评评分当氛围判断。

---

## 平台可用性（实测，2026-09 复查）

| 平台 | 状态 | 备注 |
|---|---|---|
| 搜索引擎 · 携程攻略 · 文旅厅官网 · 新闻 | ✅ 可用 | 官方信息优先 |
| 小红书 | ✅ 可用 | 2026-09 复查，此前的 IP 风控已解除 |
| 大众点评 | ⚠️ 可能需登录 | 需登录态时走浏览器 CDP 抓 |
| 马蜂窝 | ❌ WAF 拦截 | — |
| 十六番 | ❌ APP 独占 | — |

> 平台状态会变（风控、登录策略、改版），属**时效性信息**：用之前先探一次，并把复查日期记下来。

**信息源优先级**：

1. 官方景点官网、票务页、机场页、交通页
2. 餐厅：大众点评 + 小红书
3. 社媒种草、游记——**只补感觉，不能当营业时间等事实**

---

## 一、大众点评

### 前提

- Chrome 已登录 `dianping.com`
- 已安装 OpenCLI Browser Bridge 扩展
- 优先使用 PC 站；移动站对非移动 UA 限制较多

### 搜索餐厅

**围绕当天主区域搜索，不搜泛词。**

```bash
opencli dianping search "<keyword>" --city <name-or-id> --limit <n> -f json
# 例：opencli dianping search "银座 午餐" --city 东京 --limit 5 -f json
```

`--city` 可用中文、拼音或点评 cityId；省略时使用当前 cookie 里的城市。

### 查店铺详情

```bash
opencli dianping shop <shop_id> -f json
opencli dianping detail <shop_id> -f json
```

### 判断标准

优先看：`rating`（基础稳定性）、`reviews`（评价量，太少说明信号弱）、`price`（是否合预算）、
`cuisine`（是否适合这顿饭）、`district`（是否落在当天区域），
以及评价关键词：排队、踩雷、服务、游客店、性价比、是否值得专门去。

> **不要为了高分店扭曲路线。** 餐厅默认是当天区域里的补给点；
> 只有预约餐、强目的餐、用户明确指定的店，才允许成为路线锚点。

---

## 二、小红书

`agent-reach` 的小红书通道不稳定，稳定方案是 **OpenCLI + Chrome CDP**。

### 安装

```bash
npm install -g @jackwener/opencli
```

验证：`opencli --version`（应 1.7.0+）、`opencli doctor`（检查 daemon / extension / Chrome 连通性）。

**Browser Bridge 扩展**：从 [Releases](https://github.com/jackwener/OpenCLI/releases) 下载 zip →
解压到 `~/.opencli/extensions/opencli-extension` →
Chrome 打开 `chrome://extensions` → 开发者模式 → 加载已解压的扩展程序。

> OpenCLI 自带 `opencli xiaohongshu search` 命令，但它依赖 Browser Bridge + daemon 全部在线，
> 实测不一定稳定。**不稳定时走下面的 CDP 直连方案**，更可靠。

### 最小流程

**Step 1 · 启动可调试 Chrome**

```bash
chrome --user-data-dir=/tmp/opencli-chrome-cdp --profile-directory=Default \
  --remote-debugging-port=9223 'https://www.xiaohongshu.com/explore'
```

用单独的 `user-data-dir`，开调试口，先打开小红书。

**Step 2 · CDPBridge 连接**

```js
import { CDPBridge } from '<opencli>/dist/src/browser/cdp.js';
const bridge = new CDPBridge();
const page = await bridge.connect({ cdpEndpoint: 'http://127.0.0.1:9223', timeout: 10 });
```

**Step 3 · 搜索——直接进路由**

> ⚠️ **不要模拟输入框。** 小红书前端有双 input、透明 input、联想层、风控逻辑，
> 模拟输入会"假成功"。**直接导航到搜索结果页**：
> `https://www.xiaohongshu.com/search_result?keyword=<encoded>`

**Step 4 · 拦截搜索 API**

```
POST https://edith.xiaohongshu.com/api/sns/web/v1/search/notes
```

请求体：`{keyword, page, page_size, sort:"general", note_type:0}`
返回：笔记 id、xsec_token、标题、作者、点赞、收藏、评论。

**第一轮筛选不需要开详情页**——搜索前排结果就够判断信号强弱。

**Step 5 · 详情页提取**

拼 URL：`https://www.xiaohongshu.com/explore/<id>?xsec_token=<token>&xsec_source=`

```js
document.querySelector('#detail-title')?.innerText      // 标题
document.querySelector('#detail-desc')?.innerText       // 正文
document.querySelector('.author-container .username')?.innerText  // 作者
```

找不到时退回 `document.body.innerText.slice(0, 2000)`。

**Step 6 · 两段式流程**

1. 搜索结果页抓前 10–20 条
2. 只开最相关的 2–3 条详情页

好处：快、不容易被风控、先判断信号强弱、写进 `.md` 更干净。

---

## 三、筛选标准

### 保留（真店信号）

- 店名明确、地址明确、菜品明确
- 有自己体验
- 高频词在多条笔记里重复出现

### 不保留

- 泛区域合集里顺手带一句
- 标题写别的、正文才顺手提店
- 明显搬运

### 能帮决策的信息（优先）

- 要不要排队
- 是主餐还是收尾
- 更适合白天还是晚上
- 更像打卡还是更像稳饭
- 容不容易踩空

**不优先**：纯情绪表达、漂亮但没用的形容、重复三遍的"氛围很好"。

---

## 四、写回格式

每顿饭只保留 2–3 个候选，只留一层结论，不搬原文：

```md
午餐区域：<区域>
主推：店名 A
- 大众点评：评分稳定，评价量够，适合午餐，不需要专门绕路
- 小红书：近期反馈氛围好，拍照友好

备选：店名 B
- 大众点评：离地铁近，排队风险低
- 小红书：更像工作日简餐
```

小红书侧只留：**店名 + 一条代表笔记链接 + 两三句压缩判断**。

---

## 五、常见坑

- 只按评分选店，不看它是否在当天区域
- 为了一家店反向规划半天路线
- 把小红书种草当成餐厅硬口碑
- 忽略排队、预约和营业时间（**排队时长要算进当天时间预算**）
- 搜索词太泛，得到一堆游客店
- 用社媒内容当营业时间等事实
- **fetch 直接调接口可能被拦**（返回 `code:300011` 要求切换账号）→ 最稳是走真实页面 + CDP 抓响应
- **搜索结果混地区内容**——不是噪音，能看出店在片区里的角色，但不能直接当单店口碑
