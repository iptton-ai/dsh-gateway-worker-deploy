# OPERATIONS —— CF Worker 网关 + dsh-mobile 配置与操作总览

> 面向已部署(或准备部署)`dsh-gateway-worker` + `dsh-mobile` 插件的用户。
> 每项操作给 **UI 手工** 与 **命令行** 两种方案;需要用户提供凭证的地方用 🔑 标出。
> 标注约定:🔑**WRANGLER** = 需 `npx wrangler login`(浏览器授权);
> 🔑**TOKEN** = 需用户提供 [API Token](https://dash.cloudflare.com/profile/api-tokens);
> 🔑**浏览器** = 需用户在浏览器点一次授权/操作。

## 0. 形态与信任模型(一张图)

```
手机 App ─https→ 网关域名(Worker,配对/令牌防线)
                    │ fetch + Access 凭证(🔑 只有 Worker 持有)
                    ▼
             隧道主机名(公网入口,Access 把守)←─ cloudflared(dsh-mobile 插件随 dsh web 拉起)─ Mac :3080
```

三把钥匙,各管一段,**只有 ADMIN_KEY 会出现在 Mac 上**:

| 密钥 | 持有者 | 作用 | 填写位置 |
|---|---|---|---|
| `JWT_SECRET` | 仅 Worker | 签发/验证手机令牌 | 部署时一次 |
| `ADMIN_KEY` | Worker(验证)+ Mac 插件(出示) | 管理面信任根(claim/状态/吊销) | 部署时 + dsh-mobile「管理密钥」栏(**两处必须逐字一致**) |
| `CF_ACCESS_CLIENT_SECRET` | 仅 Worker | 穿 Access 门(隧道入口防线,fail-closed 必填) | 部署时一次 |

## 1. CF Worker 配置项总表(易错重灾区)

| 变量 | 类型 | 必填 | 改动:UI 手工 | 改动:命令行 |
|---|---|---|---|---|
| `JWT_SECRET` | secret | ✓ | Worker → Settings → Variables and Secrets → Type=Secret | 🔑**WRANGLER** `npx wrangler secret put JWT_SECRET --name <worker名>` |
| `ADMIN_KEY` | secret | ✓ | 同上(改后 dsh-mobile webui 也要同步换) | 🔑**WRANGLER** `npx wrangler secret put ADMIN_KEY --name <worker名>` |
| `CF_ACCESS_CLIENT_SECRET` | secret | ✓ | 同上 | 🔑**WRANGLER** `npx wrangler secret put CF_ACCESS_CLIENT_SECRET --name <worker名>` |
| `CF_ACCESS_CLIENT_ID` | var | ✓ | Settings → Variables(Type=Text) | 只能随部署生效:改 `-deploy` 仓 `wrangler.jsonc` → push(CI 重部署) |
| `TUNNEL_HOST` | var | ✓ | 同上 | 同上 |
| `MAX_UPLOAD_BYTES` | var | 默认 100MiB | 同上 | 同上 |
| `TOKEN_TTL_DAYS` | var | 默认 30 | 同上 | 同上 |

**⚠️ vars 与 secrets 的持久性差异(最易踩的坑)**:

- **secrets** 与部署解耦,`secret put` / dashboard 改完即持久,CI 重部署不会动它;
- **vars(明文)** 的真源是 `-deploy` 仓库的 `wrangler.jsonc` —— **CI 每次部署都会用它覆盖 dashboard 上的同名手改**。在 dashboard 改了 var,必须同步改 `-deploy` 仓的 `wrangler.jsonc`,否则下次 push 打回原值;
- **勿在主仓本地 `wrangler deploy`**:主仓 `wrangler.jsonc` 的 vars 是空占位,会把线上真值覆盖掉。要 CLI 部署,用 `-deploy` 仓。
- `--name` 别忘:部署实例名是 `<repo>-deploy`(如 `dsh-gateway-worker-deploy`),主仓配置里写的是 `dsh-gateway-worker`。

## 2. 一次性准备(按序;每步给双方案)

### ① 部署 Worker
- UI:README 的 Deploy to Cloudflare 按钮(部署界面逐项填 §1 的变量)。
- CLI:`git clone` 本仓 → 改 `wrangler.jsonc`(vars + routes)→ 🔑**WRANGLER** `npx wrangler deploy`。
- 代理:把 `AGENT-DEPLOY.md` 丢给任何 AI 代理(它会按本文档流程走,🔑TOKEN 处会向你要)。

### ② 绑定网关自定义域名(workers.dev 不可靠,必做)
- UI:Worker → Settings → Domains & Routes → Add → Custom domain。
- CLI:改 `-deploy` 仓 `wrangler.jsonc` 加 `"routes":[{"pattern":"<域名>","custom_domain":true}]` → push(CI 部署即绑)。
  (直接调 API 需要 🔑**TOKEN**,wrangler OAuth 对该写端点会被拒。)

### ③ cloudflared 隧道三件套(用 dsh-mobile 插件的**只能走 CLI**)
> 插件以「本地管理隧道」方式运行(config.yml + 凭证文件,dsh web 启停);
> dashboard 创建的是「远程管理隧道」(ingress 存云端),与插件不兼容,只适用于
> 不用插件、`cloudflared service install` 常驻的场景(见 AGENT-DEPLOY)。

```bash
cloudflared tunnel login                  # 🔑浏览器:点 Authorize(进程会超时,没弹就重跑)
cloudflared tunnel create dsh-gateway     # 记下 UUID;凭证落 ~/.cloudflared/<uuid>.json
cloudflared tunnel route dns dsh-gateway <隧道主机名>   # 建公网 DNS(CNAME)
```
config.yml 不用手写 —— 插件每次随 dsh web 启动按运行时端口自动生成(含 Host 改写)。

### ④ Access(service token + self-hosted 应用)—— fail-closed 必配
- UI(推荐,5 分钟):
  1. [Zero Trust](https://one.dash.cloudflare.com/) → Access → **Service Tokens → Create**(Secret 只显示一次);
  2. ID 填 Worker var `CF_ACCESS_CLIENT_ID`,SECRET 用 `secret put`/dashboard 填入;
  3. Access → **Applications → Add → Self-hosted**:域名 = 隧道主机名;策略 Action = **Service Auth**,Include = 该 token。
- CLI(🔑**TOKEN**,权限要齐:**Access: Apps and Policies Edit + Access: Service Tokens Read**(建策略要读 token 清单拿 UUID;只给 Edit 不够,实测会卡在列表)):
  ```bash
  # 拿 service token UUID
  curl -s "$API/accounts/$ACC/cfd_access_service_tokens" -H "Authorization: Bearer $CFUT"
  # 建应用(策略内联)
  curl -s -X POST "$API/accounts/$ACC/access/apps" -H "Authorization: Bearer $CFUT" \
    -H 'content-type: application/json' -d '{"name":"dsh-gateway tunnel","type":"selfhosted",
      "domain":"<隧道主机名>","policies":[{"name":"allow-gateway-worker","precedence":1,
      "decision":"service_auth","include":[{"service_token":{"id":"<UUID>"}}]}]}'
  ```
- 验证:`curl -sI https://<隧道主机名>/` → **302**(被 Access 拦 = 门装好了);
  `curl https://<网关域名>/healthz` → `"access_protected":true`。

### ⑤ dsh-mobile 插件(Mac 侧)
`~/.dsh/profiles/web/cordis.patch.yml` 的 dsh-mobile config(可加 `DSH_MOBILE_*` 环境变量覆盖):

| 键 | 说明 |
|---|---|
| `gateway` | Worker 网关地址(如 `https://dsh.example.com`) |
| `cfTunnelId` | ③ 创建的隧道 UUID |
| `cfHostname` | 隧道公网主机名(= `TUNNEL_HOST` = Access 应用域名) |
| `publicUrl` | `<gateway>/pair`(二维码落地页) |
| `target`/`remotePort`/`sockDir`/`adminPort` | Rust 形态专用;保留则两形态并存(迁移期) |

- **ADMIN_KEY 不进配置文件**:重启 dsh web 后在「📱 移动接入」弹窗的「管理密钥」栏填(持久化,保存即生效;留空保存=清除);
- 重启 `dsh web` → 插件自动拉起 cloudflared → webui 点「配对手机」。

## 3. 日常操作对照

| 操作 | UI | CLI |
|---|---|---|
| 配对手机 | 「移动接入」弹窗 → 配对手机(扫码) | `GW=<网关> ADMIN_KEY=<key> node scripts/pair.mjs <10位码>` |
| 设备清单/吊销 | 同弹窗(吊销按钮) | `node scripts/revoke.mjs --list` / `revoke.mjs <jti>` |
| 健康检查 | — | `curl https://<网关>/healthz`(`upstream`=隧道,`access_protected`=Access 凭证) |
| 换 ADMIN_KEY | Worker Settings 改 secret + webui「管理密钥」重填 | 🔑**WRANGLER** `secret put ADMIN_KEY --name …` + webui 重填 |
| 换隧道主机名 | 改 DNS/隧道 + Worker `TUNNEL_HOST` var + 插件 `cfHostname` | 同左(cloudflared `route dns` 建) |
| 查构建日志 | dashboard → Worker → Deployments(🔑唯一途径;wrangler OAuth 与 Builds API 均不可) | 重启 ZCode 后可走 cloudflare-builds MCP |
| 升级 Worker 代码 | `-deploy` 仓 `git pull` 上游 → push 触发 CI | 同左(一条命令) |

## 4. 凭证边界速查(实测)

| 凭证 | 能做 | 不能做 |
|---|---|---|
| 🔑 wrangler login(OAuth) | deploy、`secret put`、读账户/Worker/域名清单 | Builds 日志、Access API、绑域名等写 API |
| 🔑 API Token(`cfut_…`) | 按赋权(Access/Workers/DNS…) | 权限外一律 10000 |
| cloudflared cert.pem | 建隧道/route dns | Access、Worker 配置 |
| ADMIN_KEY | 网关管理面(claim/状态/吊销/QR) | 无其它权限(专钥专用) |
