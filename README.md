# llm-relay

把你自建的（或你有权使用的）大模型服务（Claude / Anthropic Messages 协议）安全地提供给自己的设备使用。服务部署在家里局域网，家里无需开放任何入站端口：请求经过网关和公网机器，再穿过家里主动建立的 TLS 隧道回到家里，并按路径区分不同的服务。

> **合规使用**：只把本项目用于你自己拥有或已获授权使用的服务和 API Key，并遵守模型服务商的使用条款以及你所在组织的网络与安全政策。

## 拓扑

```
客户端 ──①HTTPS──► 网关 ──②HTTPS :443──► 公网机器 Caddy
                                          ├─ /~!frp ──► frps 127.0.0.1:7000      隧道控制
                                          ├─ /llm/* ──► frps 127.0.0.1:18000 ─┐  校验中转 Key + 脱敏
                                          └─ 其他   ──► 404                     │
                                                                  ③ wss 隧道（TLS） │ 家里主动连公网 443
                                                                                ▼
                         大模型服务 ◄──④── 家里 Caddy 127.0.0.1:18080 ◄── frpc
                                          注入真实 Key
```

| 跳 | 加密 | 认证 |
|---|---|---|
| ① 客户端 → 网关 | 网关的证书 | 中转 Key（在公网 Caddy 校验） |
| ② 网关 → 公网 Caddy | Let's Encrypt 正式证书 | 中转 Key |
| ③ frpc → 公网 Caddy | wss（TLS），frpc 会校验证书 | frp token |
| ④ 家里 Caddy → 大模型服务 | 局域网服务用 HTTP；外部服务用 HTTPS（会校验对方证书） | 真实 Key |

公网机器上的 frps 和映射端口都只监听本机，对外只开放 443。家里的 Caddy 也只监听本机。

**请求头清理**：公网 Caddy 转发前不会把代理头（`X-Forwarded-*`、`X-Real-IP`、`Forwarded`、`Via`）和中转 Key 转给后端，响应里也会去掉 `Server` 和 `Via`。家里 Caddy 只加上真实 Key。这样中转 Key 不会泄露给后端，真实 Key 也不会出现在客户端和公网机器上。注意这只覆盖 IP 和代理头，`User-Agent` 等客户端请求头仍会原样转发。

## 目录

```
public-server/     公网机器：frps + Caddy
home-relay/        家里中转机：frpc + Caddy
gateway-example/   网关转发示例（Nginx）
```

密钥只写在各机器本地的 `.env` 文件里，不提交到 git。`docker-compose.yml` 只把每个容器需要的变量传进去，必填项留空时 `docker compose up` 会直接报错退出。

## 部署

### 1. 公网机器

前提：`RELAY_HOST` 是公网 DNS 能解析到这台机器的域名，并且 443 端口能从互联网访问。

```bash
cd public-server
cp .env.example .env && chmod 600 .env && vi .env    # FRP_AUTH_TOKEN、LLM_RELAY_KEY 用 openssl rand -hex 32 生成
docker compose up -d
docker compose logs -f caddy       # 出现 certificate obtained successfully 就说明证书申请成功了
```

安全组只放行 **443**。

### 2. 家里中转机

```bash
cd home-relay
cp .env.example .env && chmod 600 .env && vi .env
docker compose up -d
docker compose logs -f frpc        # 出现 login to server success 就说明隧道建好了
```

### 3. 网关

参考 `gateway-example/nginx-llm.conf` 配置转发。

### 4. 验证

```bash
curl -N https://<网关域名>/llm/v1/messages \
  -H "x-api-key: <LLM_RELAY_KEY>" \
  -H "anthropic-version: 2023-06-01" -H "content-type: application/json" \
  -d '{"model":"<MODEL>","max_tokens":100,"stream":true,"messages":[{"role":"user","content":"你好"}]}'
```

## 客户端

```bash
export ANTHROPIC_BASE_URL="https://<网关域名>/llm"
export ANTHROPIC_AUTH_TOKEN="<LLM_RELAY_KEY>"
export ANTHROPIC_MODEL="<MODEL>"    # 如果后端的模型名和官方不同
```

## 新增服务（以 `/nas/*` 为例）

1. `home-relay/frpc.toml`：新增一个 `[[proxies]]`，`remotePort = 18001`，指向 NAS 的地址。
2. `public-server/frps.toml`：在 `allowPorts` 里追加 `{ single = 18001 }`。
3. `public-server/Caddyfile`：复制 `handle_path /llm/*` 整块，路径改成 `/nas/*`、端口改成 18001，鉴权改成 `import guard nas {$NAS_RELAY_KEY}`；然后在 `.env` 里加上 `NAS_RELAY_KEY`，并在 `docker-compose.yml` 的 caddy `environment` 里加上 `NAS_RELAY_KEY: ${NAS_RELAY_KEY:?…}`。
4. 网关：新增 `/nas/` 的转发规则。

注意：`guard` 要求每个请求都带上中转 Key，所以适合 API 类服务。`sanitize` 会删掉 `Authorization` 和 `X-Api-Key`，如果新服务自己也要用这两个请求头，就不要引入它。

## 常见问题

| 现象 | 排查 |
|---|---|
| 401 | 中转 Key 不对 |
| 404 | URL 少了 `/llm` 前缀 |
| 502 | 隧道没建好（看 frpc 日志），或者家里访问不到 `LLM_UPSTREAM` |
| 后端返回 401 | `BACKEND_AUTH_HEADER` 或 `BACKEND_AUTH_VALUE` 不对 |
| 证书申请失败 | `RELAY_HOST` 没有解析到这台机器，或者 443 端口从公网访问不到 |
| frpc 报 `x509` | 两边的 `RELAY_HOST` 不一致，或者证书还没申请下来 |
| 网关报证书错误 | `proxy_ssl_trusted_certificate` 没有指向系统证书库 |
| 流式输出一次性返回 | 网关没有关闭缓冲 |

`caddy_data/` 里保存着证书和 ACME 账号，保留它可以避免每次重启都重新申请证书。

两边的 Caddy 都关闭了管理 API（`admin off`），修改 Caddyfile 或 `.env` 后用 `docker compose up -d --force-recreate caddy` 生效，`caddy reload` 不可用。

## 安全须知

- **密钥强度**：`FRP_AUTH_TOKEN` 和 `LLM_RELAY_KEY` 都用 `openssl rand -hex 32` 生成，不要用短密码。两者留空时 compose 会拒绝启动；不要绕过这个检查，否则公网入口会变成任何人都能用的开放代理，或者 frp 不做鉴权。
- **中转 Key 等同于你的额度**：中转 Key 是共享且长期有效的，没有限流和配额，拿到它的人可以消耗你的后端额度。只发给你自己的设备，怀疑泄露时立即更换（改 `.env` 后重建容器），并建议在模型服务商处设置用量上限。
- **保护 `.env`**：`chmod 600 .env`，不要提交、截图或贴到聊天里；备份时注意 `.env` 中是明文密钥。
- **避免密钥进入 shell 历史**：上面的 `curl -H` 和 `export` 示例会把 Key 留在 shell 历史或 dotfile 里，可以用 `read -s LLM_RELAY_KEY` 读入，或写到 Claude Code 的 `settings.json`（权限设为仅自己可读）。
- **家里中转机的信任边界**：家里 Caddy 在 `127.0.0.1:18080` 上不做鉴权，中转机本机的任何进程都能借它使用真实 Key，请只在你信任的机器上部署。
- **只开放 443**：公网机器的安全组只放行 443；frps 控制端口和映射端口都只监听 `127.0.0.1`，`allowPorts` 只放行实际用到的端口。

## 术语表

| 术语 | 说明 |
|---|---|
| frp（frps / frpc） | 内网穿透工具。内网的 frpc 主动连接公网的 frps，建立隧道 |
| wss | 基于 TLS 的 WebSocket，这里让隧道走 443 端口 |
| Caddy | 反向代理，负责 TLS、按路径分发、鉴权和请求头清理 |
| Let's Encrypt | 免费的公共证书颁发机构，签发的证书被所有系统信任，Caddy 会自动申请和续期 |
| TLS-ALPN-01 | Let's Encrypt 的一种验证方式，通过 443 端口确认域名归你所有 |
| SNI | TLS 握手时，客户端告诉服务端自己要访问的主机名 |
| SSE | Server-Sent Events，大模型流式输出用的协议 |

## 参考文献

- frp：https://github.com/fatedier/frp
- Caddy：https://caddyserver.com/docs/caddyfile
- Anthropic Messages API：https://docs.anthropic.com/en/api/messages
- Claude Code 设置：https://docs.anthropic.com/en/docs/claude-code/settings
