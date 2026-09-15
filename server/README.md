# Throne Race 在线对战服务器

Cloudflare **Worker + Durable Object** 房间中继。架构是「房主权威」：

- 每个房间一个 Durable Object，只负责座位分配、花名册、开局和消息转发，**不理解游戏规则**；
- 游戏逻辑（走子/放墙校验、留路 BFS、AI 补位、道具）全部运行在**房主的浏览器**里；
- 其他玩家的操作经服务器转给房主校验执行，房主把状态快照广播给所有人。

线上地址：**`https://throne-race-online.1kb.ren`**（部署在 1kb.ren 所在的 Cloudflare 账号，用官方 Custom Domain 绑定，国内可直连）。备用地址 `https://throne-race-online.breadbot86.workers.dev`（workers.dev 在大陆被墙，需代理；部署在另一个账号）。

免费版 Workers 完全够用：每天 10 万次请求（WebSocket 收 100 条消息才计 1 次）、13,000 GB-s 时长，且发出的消息免费。一局棋只有几百条小消息。

## 部署（约 2 分钟）

需要目标账号的凭证。两种方式任选：

```bash
# 方式 A：wrangler OAuth 登录（浏览器点一次授权）
npx wrangler login

# 方式 B：远程授权 / API Token
#   CLOUDFLARE_API_TOKEN=<token> npx wrangler deploy
```

```bash
cd server
npx wrangler deploy
```

`wrangler.toml` 里已固定 `account_id` 和 Custom Domain 路由（`throne-race-online.1kb.ren`），部署时 DNS 记录与证书由 Cloudflare 自动创建。

> ⚠️ 经验教训：**跨账号把橙云 CNAME 指向 workers.dev 会被 Cloudflare 拒绝（error 1014，CNAME Cross-User Banned）**。域名在别的账号时，正确做法就是把 Worker 部署到域名所在账号再用 Custom Domain，而不是加解析指向 workers.dev。

前端地址在 `index.html` 顶部 `NET_WS_DEFAULT`，改完发布到 GitHub Pages 即生效。

## 本地联调 / 测试

```bash
cd server && npx wrangler dev --port 8787          # 终端 1：本地跑服务器
node server/test.e2e.mjs ws://127.0.0.1:8787/api/room   # 终端 2：协议端到端测试（18 项断言）

# 浏览器联调：临时用 ?ws= 参数指向本地服务器，无需改代码
open "http://localhost:8080/index.html?ws=ws://127.0.0.1:8787/api/room"
```

## 协议速查

| 方向 | 消息 | 说明 |
|---|---|---|
| C→S | `{t:'hello', name}` | 连上后报名字，服务器分配座位（加入顺序 = 蓝绿红橙） |
| S→C | `{t:'welcome', seat, isHost, code, players}` | 首位加入者为房主 |
| S→C | `{t:'roster', players}` / `{t:'left', seat}` | 大厅与对局中的成员变动 |
| C→S | `{t:'start', settings}` | 仅房主；`settings = {sync, watch, size, walls, items}`，`watch` 为观战局（全员 AI 围观） |
| S→C | `{t:'start', settings, seats}` | 广播开局；未加入的座位由 AI 补位（跑在房主端） |
| C→S | `{t:'act', a}` | 玩家操作意图（move/wall/smash），仅转给房主校验 |
| H→S | `{t:'state', s, ev}` | 仅房主；权威状态快照 + 事件（音效/特效/提示），广播给其他人 |
| S→C | `{t:'host-left'}` | 房主离开 = 房间解散 |

玩家掉线时其座位自动转给 AI（房主端接管）；房间生命周期 = 房主在线时长，无持久化。
