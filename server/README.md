# Throne Race 在线对战服务器

Cloudflare **Worker + Durable Object** 房间中继。架构是「房主权威」：

- 每个房间一个 Durable Object，只负责座位分配、花名册、开局和消息转发，**不理解游戏规则**；
- 游戏逻辑（走子/放墙校验、留路 BFS、AI 补位、道具）全部跑在**房主的浏览器**里；
- 其他玩家的操作经服务器转给房主校验执行，房主把状态快照广播给所有人。

免费版 Workers 完全够用：每天 10 万次请求（WebSocket 收 100 条消息才计 1 次）、13,000 GB-s 时长，且发出的消息免费。一局棋只有几百条小消息。

## 部署（约 2 分钟）

```bash
cd server
npx wrangler login     # 浏览器弹出 Cloudflare 授权，用部署 Worker 的那个账号
npx wrangler deploy
```

部署成功会输出形如 `https://throne-race-online.<你的子域>.workers.dev` 的地址。

然后在 `index.html` 里把地址填进顶部常量并发布到 GitHub Pages：

```js
const NET_WS_DEFAULT = 'wss://throne-race-online.<你的子域>.workers.dev/api/room';
```

## 绑定自己的域名（域名在另一个 Cloudflare 账号时）

Worker 所在账号无需任何操作。去**域名所在的账号**：

1. 该域名 → **DNS → 添加记录**
   - 类型：`CNAME`
   - 名称：`play`（任意子域）
   - 目标：`throne-race-online.<你的子域>.workers.dev`
   - 代理状态：**已代理（橙色云）**
2. 该域名 → **SSL/TLS → 概述 → 加密模式改为「完整 (Full)」**
   （默认的「灵活」会用 HTTP 回源，WebSocket 会握手失败——这步不做联机连不上。）
3. 把 `NET_WS_DEFAULT` 改成 `wss://play.你的域名.com/api/room`，重新发布。

之后 `workers.dev` 原始地址和自定义域名都能用；国内建议走自定义域名（`workers.dev` 在大陆基本被污染）。

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
| C→S | `{t:'start', settings}` | 仅房主；`settings = {sync, size, walls, items}` |
| S→C | `{t:'start', settings, seats}` | 广播开局；未加入的座位由 AI 补位（跑在房主端） |
| C→S | `{t:'act', a}` | 玩家操作意图（move/wall/smash），仅转给房主校验 |
| H→S | `{t:'state', s, ev}` | 仅房主；权威状态快照 + 事件（音效/特效/提示），广播给其他人 |
| S→C | `{t:'host-left'}` | 房主离开 = 房间解散 |

玩家掉线时其座位自动转给 AI（房主端接管）；房间生命周期 = 房主在线时长，无持久化。
