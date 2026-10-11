# PipePipe B 站风控 / 组刷新漏抓 设计规格

- 日期：2026-10-11（v2，按"最小改动 / 可 upstream"约束重写）
- 范围：B 站（BiliBili）风控触发率、订阅组刷新漏抓、刷新失败后无法重试
- 不在范围内：视频下载（等复现，见 §10）；身份持久化 / 接口降级 / 412 验证码（见 §6 后续）
- 基线：父仓库 `v5.4.0`（`7349b0f`），submodule 已同步到 `PipePipeClient@c2a166f7d`、`PipePipeExtractor@c68e10e2e`

---

## 0. 改动原则（本轮的硬约束）

1. **最小改动、可 upstream**：不新增跨模块抽象、不改公共接口签名、不重排既有类结构。单个改动尽量是"几行 ~ 几十行"。
2. 每个 S 项都要能单独成一个 PR。
3. 不引入新依赖、不引入新的构建配置。
4. 任何对 B 站的实测必须有请求上限 + 间隔，遇非 0 立即停止。

---

## 1. 目标与成功标准

| # | 目标 | 成功标准 |
|---|---|---|
| G1 | 组刷新不再打被风控的 web 投稿列表接口 | 刷新期间 `api.bilibili.com/x/space/...` 请求数为 **0** |
| G2 | 组刷新不再漏 UP 的新投稿 | UP 发新视频后手动刷新，**最长 15 秒**后必然出现在组里；失败过的订阅**立即**可重试 |
| G3 | 刷新失败后刷新入口仍然可用 | 连续两次"失败 → 重试"都能真正发出请求 |
| G4 | 降低单轮请求量 | 每 UP 请求数从 **6 降到 3** |

---

## 2. 证据

### 2.1 实测（2026-10-11，本机 IP，约 12 个有界请求）

| 请求目标 | 结果 |
|---|---|
| `x/space/wbi/arc/search`（**Web API，当前默认**） | 第 1 次 `code=0`，**第 2 次即 412 HTML** |
| `x/series/recArchivesByKeywords`（Search API） | 412 HTML |
| `app.bilibili.com/x/v2/space/archive/cursor`（**App 客户端 API**） | `code=0`，20 条/页，字段齐全 |
| `x/web-interface/wbi/view`、`x/web-interface/card` | `code=0`（在 412 之后立刻复测） |
| `ExClimbWuzhi` 身份激活 | 调用成功 |

结论：**412 是接口级的**，不是 IP 级、也不是身份级。被拦的是 web 系投稿列表接口。

### 2.2 代码证据

| 结论 | 位置 |
|---|---|
| 默认走 Web API（即被拦的那个） | `BilibiliService.java:541` `userVideoApiMode = USER_VIDEO_API_MODE_WEB` |
| 412 时只换设备重试 1 次，然后抛异常 | `BilibiliChannelExtractor.java:542-586` |
| 每 UP 重复抓一遍同样的接口 | `FeedLoadManager.kt:174-215` + `BilibiliChannelTabExtractor.java:35-42` |
| 每服务限速是读-改-写竞态 | `FeedLoadManager.kt:126-141` |
| 手动刷新不带 force，受 300s 阈值限制 | `FeedFragment.kt:1059-1067`、`FeedLoadService.kt:92-93` |
| **失败过的订阅不会被阈值挡住** | `FeedDAO.kt:281`：`WHERE lu.last_updated IS NULL OR lu.last_updated < :outdatedThreshold`；失败时 `FeedDatabaseManager.kt:78-79` 把 `last_updated` 置 null |
| 抓取成功即无条件置 `last_updated = now` | `FeedDatabaseManager.kt:116-118` |
| **`loadingDisposable` 从不清空**，服务存活期间的 start 命令被静默丢弃 | `FeedLoadService.kt:85-87`（早退）、`:107-117`（`disposeAll`/`stopService` 不置 null）、`:98-103` |
| 错误面板是 layout 最后一个子 View，会盖住刷新入口 | `fragment_feed.xml:143-144`、`FeedFragment.kt:802-810` |

---

### 2.3 真代码级验证（本次实际执行）

本机没有完整构建环境（Gradle wrapper 要下 9.5.1、toolchain 要 JDK 25、`~/.gradle` 不可写），所以改用 `javac` + 手工拉取依赖（含用 wire-compiler 生成的 protobuf 类），编译了 extractor 的全部源码，并实现了一个最小 `Downloader`，**直接调用 PipePipe 自己的 `ChannelExtractor` 路径**做 A/B：

| 模式 | page1 | page2 | 结果 |
|---|---|---|---|
| **CLIENT（App API，S1a 拟改为默认）** | 20 条 | 19 条，与 page1 **零重叠** | ✅ 分页正确；连续 3 次运行全部成功 |
| **WEB（当前默认）** | — | — | ❌ `ParsingException: BiliBili blocked us, we retried 2 times`（412） |

同一 IP、同一时刻、同一份伪造设备与 cookie，**CLIENT 通、WEB 被拦**。S1a 已端到端验证可行，且 CLIENT 模式的游标（`goto`/`param`）分页确实工作。

边界说明：这是 1 个 UP、1 个 IP、约 40 分钟内 3 次运行的结果，不是严谨压测。WEB **并非永久被拦**（前两次运行它也成功返回了 25 条），更准确的说法是：**两个接口各有独立的请求预算，web 侧预算会被逐渐耗尽，app 侧宽得多**。这修正了 §2.1 里"接口被拦"的过强表述。

复现方式：`.verify/Verify.java`（已通过 `.git/info/exclude` 排除，不进 git）。

---

### 2.4 取数方式选型实测（2026-10-11）

同一份伪造身份、同一批 UP、**交错执行**以避免顺序偏差（每模式 12 次请求，间隔 1.5s，失败也继续计次）：

| 方法 | 端点 | 成功 | 首次失败 | 平均延迟 |
|---|---|---|---|---|
| **CLIENT（App 接口）** | `app.bilibili.com/x/v2/space/archive/cursor` | **12/12** | 从未失败 | 0.11s |
| SEARCH | `api.bilibili.com/x/series/recArchivesByKeywords` | 7/12 | 第 2 个请求 | 0.05s |
| WEB（原默认） | `api.bilibili.com/x/space/wbi/arc/search` | **3/12** | **第 1 个请求** | 0.14s |
| 对照 `wbi/view` | — | ✅ 测试首尾均正常 | — | — |

**持续压测**：CLIENT 连打 24 次（1 请求/秒）→ **24/24 全成功，零失败**；WEB 单发即 `code -352`，**等待 20 秒后恢复** —— 说明 web 侧是**短窗口限流**（约 20 秒冷却），不是永久封禁。这正好解释了用户"点一遍正常、点第二遍就出验证码"的现象。

**结论**：端点优先级 `CLIENT → SEARCH → WEB`，由 `BilibiliService.USER_VIDEO_API_MODE_ORDER` 定义（commit `b3fa7254`）。原实现是 `CLIENT → WEB → SEARCH`，把最差的放在第二位。

### 2.5 端到端验证（2026-10-11，真机 华为 Mate 60 Pro）

一次「键政」组刷新（9 个 UP），用临时请求日志在真机上逐条数请求：

| 接口 | 次数 |
|---|---|
| `app.bilibili.com/x/v2/space/archive/cursor`（投稿列表） | **9** |
| `api.bilibili.com/x/web-interface/card`（用户名片） | **9** |
| `api.live.bilibili.com/room/v1/Room/get_status_info_by_uids`（直播状态） | **9** |

**9 × 3 = 27 个请求**（修复前 9 × 6 = **54**）。重复的 tab 抓取端点 `x/polymer/web-space/seasons_series_list` **一次都没出现** → **S3a 生效**。

请求时序：相邻订阅间隔 **恰好 1.0 秒**（14:05:28.4 / 29.4 / 30.4 / … / 36.4），服务总时长 **8.45 秒**；修复前是 10.8 秒内爆发式打完 → **S3b 生效**。

数据库对比（刷新前 → 刷新后）：

| UP | 前 | 后 | |
|---|---|---|---|
| 波士顿圆脸 | 20 | 20 | = |
| 瓜熟迪落拉 | 13 | 13 | = |
| 熊猫围观 | 4 | 4 | = |
| 红色幻想乡 | 17 | 17 | = |
| BMZ账单 | 8 | **7** | 轮播条目被正常清理 |
| TomCat团座 | 18 | 18 | = |
| 双尾彗星 | 14 | **13** | 同上 |
| 燕三嘤嘤嘤 | 15 | 15 | = |
| 维兹克斯考尔 | 0 | 0 | = |

总量 1331 → 1329。逐条核对，少的 2 条都是 `LIVE_STREAM` 类型、标题为"XX的投稿视频轮播" —— 正是 `unlinkOldLivestreams` 该清理的对象。**没有一个普通视频丢失** → feed 修复生效。

### 2.6 名片按需刷新（2026-10-11）

**动机**：feed 刷新原来对每个订阅都取一次用户名片（1 请求/UP）。核对后确认：名片里 feed 真正不可替代的只有 `name`，而 `name` 在投稿列表里就有（`item.author`）；`face/fans/sign` 只用于更新订阅行元数据（`FeedUpdateInfo` 对它们都有回退：`?: subscription.avatarUrl` / `?.let`），而 feed 列表**根本不显示头像**。

**设计**：客户端把该订阅已存的名字/头像/粉丝/简介作为 hint 传进 extractor（ThreadLocal，因为多订阅并行抓取）；extractor 在拿到投稿列表后比较 `author` 与已知名字，只在下列情况才真的取名片：

1. 没有任何已知信息
2. 已知头像为空（从没取到过）
3. `author` 与已知名字不一致（UP 改名了）

跳过的那些 getter 返回**已知值本身**（不是空值），这样订阅行的写回是无变化的。

**真代码验证**（extractor harness，统计实际请求）：

| 场景 | 实际请求 |
|---|---|
| 无 hint（频道页 / 首次） | videos + **card** + live |
| hint 名字一致（feed 刷新） | videos + live ← 名片被省掉 |
| hint 名字不符（已改名） | videos + **card** + live |
| hint 无头像 | videos + **card** + live |

四个场景 `getName()` 都返回正确的频道名。

**效果**：键政组（9 UP）27 → 18 请求；212 UP 的组 636 → 424。

### 2.7 端到端验证 v2：名片按需刷新 + 100ms 间隔（真机）

| 指标 | 上一版 | 本版 | 说明 |
|---|---|---|---|
| 名片请求 | 9 | **0** | 名字未变且有头像 → 全部跳过 |
| 投稿请求 | 9 | 9 | `app.bilibili.com/x/v2/space/archive/cursor` |
| 直播请求 | 9 | 9 | `api.live.bilibili.com/.../get_status_info_by_uids` |
| **请求总数** | **27** | **18** | 原始版本为 54 |
| **刷新耗时** | **8443 ms** | **1209 ms** | 快 7 倍 |

9 个投稿请求的时间戳为 `15:42:04.275 → 15:42:05.073`，间隔正好 **100ms**，与配置一致。

数据库零回归：feed 1329 / streams 1417 前后完全一致；9 个 UP 的条目数、订阅名、头像全部不变。

### 2.8 速率上限实测（2026-10-11）

对 App 投稿接口做阶梯加压，**一失败即停**，每档后探测对照接口：

| 目标速率 | 发出 | 成功 | 实测吞吐 | 对照 `wbi/view` |
|---|---|---|---|---|
| 50 req/s × 2s | 100 | 100 | 45.4 req/s | code=0 |
| 100 req/s × 2s | 200 | 200 | 84.5 req/s | code=0 |
| 200 req/s × 2s | 400 | 400 | 102.6 req/s | code=0 |
| 300 req/s × 2s | 600 | 600 | 98.6 req/s | code=0 |

**1300 个请求、零失败，IP 未被标记。** 200/300 档的实测吞吐卡在 ~100 req/s，是测试客户端自身到顶，不是服务端限流 —— **上限未被探到**。

结论：App 接口的限流阈值比客户端任何合理行为都高出两个数量级；原来的风控完全来自 web 接口（1~2 请求即 412，冷却约 20 秒），这也正是改用 App 接口能解决问题的原因。因此 `getFeedFetchInterval()` 定为 **100ms**（10 订阅/秒），比已实测的 ≥100 req/s 低 10 倍以上。

### 2.9 修掉「只有一个视频的 UP 进主页转圈不停」（2026-10-11）

**现象**：只有 1 个视频的 UP，进频道页后视频列表下方一直转圈。

**排查**：日志显示这次进主页只有 4 个请求、之后彻底安静（无请求风暴），说明是纯 UI 卡住而非网络循环。extractor 的翻页本身正常（第 2 页返回空、`nextPage=null`）。

**根因**：App 接口的响应里本来就有权威字段 `data.has_next`：

| UP | 页 | 条数 | `has_next` |
|---|---|---|---|
| 只有 1 个视频 | page1 | 1 | **false** |
| 正常 UP | page1 | 20 | true |
| 正常 UP | page2 | 19 | true |

但 `ClientUserVideoImpl` 用的是"条目数组非空"判断 → 1 个视频也报告"有下一页" → `ChannelVideosFragment` 显示 load-more footer；而列表只有 1 条**滚不动**，滚动监听永不触发 → `loadMoreItems()` 永不调用 → **转圈永不停**。

**修复**：改用 `has_next`。harness 实测：

| UP | 修复前 | 修复后 |
|---|---|---|
| 只有 1 个视频 | `nextPage=true` → 卡死 | `nextPage=null` → 正常结束 |
| 正常 UP | 正常翻页 | 不变 |

**真机确认**：进 `维兹克斯考尔` 主页转圈消失；UI 日志里 `loadMoreItems` 零次调用、无异常。

### 2.10 翻页缓存：刷新不再漏掉滚出第一页的投稿（2026-10-11）

原来的逻辑只在"第一页为空"时才翻第二页。一个 UP 两次刷新之间发布的视频如果超过一页，滚出去的部分就被静默丢掉。

改为：**继续翻页直到某页出现数据库里已有的视频**，上限 3 页额外翻页。只对刚取回的那一页查库，避免重复查询。代价只发生在首次刷新或长期未刷。

真机（女kig 212 个订阅）：`archive/cursor` 255 次 = 212 订阅 + **43 个补翻页**。

### 2.11 女kig 分组数据对账（2026-10-11）

用 App 接口逐个订阅核对 13 周窗口内的视频：

| 结果 | 数量 |
|---|---|
| 与 B 站完全一致 | 142 |
| 双方都为空（13 周内确实没投稿） | 56 |
| DB 条目少于 B 站 | 14 |

对那 14 个逐一比对新旧：**最新视频 14/14 与 B 站完全一致，没有漏掉任何新视频**。差异全部是更旧的视频 —— DB 对高产 UP 只保留最新约 20 条（一页），更早但仍在 13 周内的没有，属于此前破坏性删除清空历史后的遗留，不是新视频丢失。

> 教训：对账脚本初版误把 `subscriptions.uid`（本地自增主键）当成 B 站 mid，得出"19 个订阅缺视频"的**错误结论**。实际两者完全不相等（226 个里 0 个相等），mid 必须从 `url` 里解析。

## 3. 本轮设计（全部为小改动）

### S1a — 默认改用 App 客户端 API（G1 / G4）

- `BilibiliService.java:541`：`USER_VIDEO_API_MODE_WEB` → `USER_VIDEO_API_MODE_CLIENT`。**一行**。
- 保留 `rotateVideoApiMode()` 与 WEB/SEARCH 作为 fallback，不动 `getVideoImpl()` 的结构。
- 验收 A1。

> App UA（`getAppHeaders()`）**不在本轮**：spike 显示 App 接口对伪造 Chrome UA 也返回 `code=0`，先单独验证"只改默认模式"的效果，避免一次改两件事。

### S3a — 去掉重复抓取（G4）

- `FeedLoadManager.kt:199-215`：遍历 fetch-enabled tab 前先判断 `tab.getUrl().equals(channelInfo.getUrl())`；同源则直接使用 `channelInfo.getRelatedItems()` 与 `channelInfo.getNextPage()`，跳过 `getChannelTab()`。
- B 站恒同源；其它服务走原路径。

### S3b — 让每服务限速真正生效（G1）

- `FeedLoadManager.kt:129-141`：把"读 `lastFetchTimePerService` + 算 wait + 写回"放进一个 `synchronized` 块后再返回 wait（不新增类）。
- `BilibiliService.java:428`：`getFeedFetchInterval()` 从 `10000` 改为常量（初值 `1000`，加注释说明可调）。

### S5a — 手动刷新改为全量，**取消 60 秒这类新增防抖**（G2）

**先回答"60 秒有必要吗"：没有。**

- 关键事实：**阈值从来不会挡住失败重试**——失败订阅的 `last_updated` 被置 null，而 SQL 是 `WHERE lu.last_updated IS NULL OR ...`（`FeedDAO.kt:281`），`IS NULL` 恒真。所以"刷新失败后重试不了"**不是**防抖造成的，是 S5c 那个独立缺陷。
- 60 秒唯一的作用是防连点，而连点在"上一轮加载仍在进行中"时已经被 `FeedLoadService.onStartCommand:85` 天然挡住了（第 2 次 start 会被丢弃）。加载本身要跑几十秒，两次全量刷新自然被隔开。
- 改动（最小）：`FeedFragment.kt:1059-1067` 的 `reloadContent()` 增加一个 extra（`EXTRA_IGNORE_OUTDATED_THRESHOLD = true`），`FeedLoadService.kt:92-93` 读出后传给已有的 `startLoading(groupId, ignoreOutdatedThreshold)` 参数。**不新增任何时间阈值、不新增类**。
- 验收 A2、A3。

> 如果实测发现"连点两次仍会触发风控"，再单独加一个 15 秒短防抖（失败订阅永不参与）。这条记在 §9 未决里，不作为本轮默认。

### S5b — `upsertAll` 不再无条件标记已更新（G2）

- `FeedDatabaseManager.upsertAll`：新增可选参数 `hasErrors: Boolean = false`；当 `itemsToInsert` 为空**且** `hasErrors` 时不写 `last_updated = now`。
- 调用点 `FeedLoadManager.kt:325` 传入 `reportableErrors.isNotEmpty()`。

### S5c — 修"刷新失败后刷新入口失灵"（G3）

两个独立缺陷，都在 `FeedLoadService.kt`：

1. `loadingDisposable` 用完不清空：`stopService()` 只 dispose，不置 null；若下一次 `startService` 落在服务实例被销毁之前，`onStartCommand` 在 `:85` 早退，**刷新静默失效**。
   → 在 `doOnSubscribe` 前后维护状态，或把 `:85` 的判断从 `loadingDisposable != null` 改为 `loadingDisposable?.isDisposed == false`。
2. 错误面板覆盖刷新入口：`error_panel` 是 `fragment_feed.xml` 的最后一个子 View，`showError()` 之后下拉刷新与刷新条都可能不可达。
   → 保证错误态下至少有一个可用入口（错误面板的 Retry 已存在，需确认它走的是 `reloadContent()` 且不被缺陷 1 挡住）。
- 验收 A4。

---

## 4. 验收清单

| 编号 | 验收项 | 方法 |
|---|---|---|
| A1 | 组刷新期间无 `api.bilibili.com/x/space/...` 请求 | logcat 打印请求 URL 计数 |
| A2 | UP 发新视频后手刷，15 秒后（或立即）出现 | 手动 e2e |
| A3 | 失败过的订阅下一次刷新立即可重试（不受防抖） | 手动 e2e |
| A4 | 连续两次"制造失败 → 点重试"都能真正发出请求 | 手动 e2e + logcat |
| A5 | 每 UP 请求数 6 → 3 | 计数 |
| A6 | 30 订阅组刷新，同 service 两次请求间隔 ≥ 配置值 | logcat 时间戳 |

---

## 5. 测试策略

现状：`PipePipeExtractor/extractor/src` 只有 `main`；`PipePipeClient/app/src` 只有 `androidTest`。两侧 build 均已配好 JUnit。

- 本轮**不新建测试目录**（与"最小改动"一致）：主要靠 logcat 断言 + 手动 e2e（§4）。
- 若 S3b 的限速逻辑改成独立可测函数，再补一个最小 JVM 单测（可选）。
- 所有对 B 站的实测：请求上限 + 间隔，遇非 0 立即停止。

---

## 6. 后续（较大改动，本轮不做）

按"不重写"的约束，以下三项**暂缓**，等 §3 的效果数据出来再评估：

| 项 | 内容 | 为什么暂缓 |
|---|---|---|
| S2 | 设备/Cookie 跨进程持久化（新增 `BilibiliIdentityStore` 接口 + 客户端实现 + 设置项） | 跨模块新接口，改动面大，且需要 upstream 接受新 API |
| S1b | API 模式选择器（冷却表 / 按调用降级），去掉全局 static 污染 | 需要重构 `getVideoImpl()` 与失败路径 |
| S4 | 412 安全验证 WebView（新 Activity + 新异常 + cookie 注入） | 成本最高；若 S1a 已让 412 变罕见，性价比待重估 |

---

## 7. 前置（已完成）

- 三个仓库已 fork 到 `BUnipendix`，并在父仓库与两个 submodule 中挂为 `fork` remote（`origin` 仍指向上游）：
  - `BUnipendix/PipePipe`、`BUnipendix/PipePipeClient`、`BUnipendix/PipePipeExtractor`
- 两个 submodule 已 `git submodule update` 到父仓库记录的 v5.4.0 指针。
- `.gitmodules` **未改动**（避免把 fork 地址写进要提交的 diff）。

---

## 8. 实施顺序

1. S1a（改默认模式）→ 验 A1、A5。
2. S3a（去重）+ S3b（限速）→ 验 A6。
3. S5a + S5b（全量刷新 + last_updated 语义）→ 验 A2、A3。
4. S5c（刷新入口失灵）→ 验 A4。

每步一个 commit，可独立 revert / 独立提 PR。

---

## 9. 未决

- `getFeedFetchInterval` 的最终取值需要在真实设备上有界压测后确定（初值 1000ms）。
- 是否需要 15 秒短防抖，取决于 S5a 落地后「连点两次」是否仍会触发风控。

## 10. 范围外 / 待复现

- **#3 无法下载视频**：v5.4.0 已含 `b3303f47`（M-CDN 过滤）、`14e111b35`（Range 探测）、`14170fc7d`（SAF 合流）三个修复，属第四个未知原因。需要复现细节（按钮灰 / 立即报错文案 / 卡在合流 / 文件不可播 + 机型 + 存储位置）。
