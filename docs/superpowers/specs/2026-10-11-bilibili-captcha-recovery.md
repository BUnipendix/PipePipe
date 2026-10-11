# B 站风控的验证码恢复（设计稿）

> 2026-10-11。这是 §4/S4 的展开：按需请求与退避都做完之后，真正被限流时的兜底手段。
> 尚未实现，先把已经查清的事实与设计记下来。

## 1. 为什么需要它

按需请求（名片、直播状态）和冷却退避只能**降低**触发风控的概率，不能让一个已经被拒的
会话恢复。真正恢复访问要么换身份（已实现），要么过一次验证码。

## 2. 两套不同的风控响应

这是实现前必须分清的，两者的恢复路径完全不同。

| 形态 | 特征 | 恢复方式 |
| --- | --- | --- |
| **JSON `-352`** | `code: -352`，`data.v_voucher`，响应头 `x-bili-gaia-vvoucher` | 文档化的 gaia 流程，**纯 API 可驱动** |
| **412 HTML 页** | `<!DOCTYPE html>`，`错误号: 412`，加载 `security.bilibili.com/static/js/412.js` | JS 自己跑极验，**必须用 WebView** |

实测遇到的是后者（见 §2.5 的真机日志）。

## 3. gaia 流程（对应 `-352`）

1. `POST https://api.bilibili.com/x/gaia-vgate/v1/register`，body 传 `v_voucher`（`csrf` 可选）
   → 返回 `type: geetest`、`token`、`geetest.gt`、`geetest.challenge`
2. 用极验完成人机验证，得到 `validate` 与 `seccode`
3. `POST https://api.bilibili.com/x/gaia-vgate/v1/validate`，body 传
   `challenge` `token` `validate` `seccode`（`csrf` 登录时必要）
   → 返回 `is_valid: 1` 与 `grisk_id`
4. 原请求：URL 加参数 `gaia_vtoken=<grisk_id>`，Cookie 加 `x-bili-gaia-vtoken=<grisk_id>`

## 4. 有效期（关键约束）

| 参数 | 有效期 | 依据 |
| --- | --- | --- |
| `v_voucher` | **一次性，且需立即使用** | 文档："同一有效 v_voucher 只能请求一次，请求完毕请立即进行验证防止过期失效" |
| geetest `token` / `challenge` | **会过期** | `validate` 返回 `100003: 验证码过期` |
| `grisk_id`（`gaia_vtoken`） | **文档未说明** | 保守处理：视为短有效期，失效即重走流程 |
| `bili_ticket` | 有明确 `created_at + ttl` | 代码已在读 `bili_ticket_expires` |

结论：`v_voucher` 拿到后必须**立即**走完 register，不能攒着、不能重试 register。
`grisk_id` 拿不到明确有效期，实现上要以"用一次，失败就重来"为准，不要缓存太久。

## 5. 不能保证成功

文档明确："不是所有风控都可以使用本方式通过 captcha 解决"。
`register` 返回的 `geetest` 若为 `null`，就是这种情况 —— 此时**不要**弹验证码给用户，
直接走退避，否则用户白过一次。

## 6. 设计

### 6.1 前台（频道页 / 播放，有界面）

复用已有的重验证机制（`ReCaptchaActivity` + WebView）：`DownloaderImpl.getCookies()`
本来就会无条件附加 recaptcha cookie，所以 WebView 过掉的关卡只要把 cookie 采下来，
后续请求自动就带上了。

### 6.2 后台/服务里（FeedLoadService，无界面）

服务里没有界面，改为**发通知**（"需要人机验证，点此完成"），点击进入验证界面。

**流程是"等用户过完再刷新"，而不是让本次刷新带着失败继续跑**：

1. 命中可过验证码的风控 → 立即把 `v_voucher` 用掉拿到 geetest（见 §4：一次性、不能攒）
2. 投出通知，刷新**挂起等待**用户完成
3. 用户完成 → 采下 `grisk_id` / `x-bili-gaia-vtoken`，**重新刷新**（把刚才失败的频道补上）
4. 用户一直没完成 → 超时（例如几分钟）后恢复原样：按退避继续，下次再说

第 3 步选择"重新刷新"而不是"从断点续传"：被风控的频道是散落的，重跑一遍最简单可靠，
而且此时身份/令牌都是新的，成功率最高。

### 6.3 不要在这一步卡死

等待必须有超时。用户把 App 切走、或者忘记点通知时，服务不能一直挂着 —— 超时后退回
§6.1 之外那条路：换身份 + 冷却退避，让后面的刷新自然补上。

## 7. 实现顺序建议

1. 先让提取器把"本次风控是否可以过验证码"判断出来（`geetest != null`），避免白弹
2. 前台：接线到现有 WebView 重验证流程
3. 后台：通知 + 点击进入验证
4. 真机验证：`card -> 412` 应为 0（按需请求已解决常规情况），验证码只在真被拒时出现
