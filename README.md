# grok2api-enhanced

`grok2api-enhanced` 是基于
[jiujiu532/grok2api](https://github.com/jiujiu532/grok2api)
的二次开发增强版，面向自托管、长期运行和私有化部署场景。

本项目保留上游 `grok2api` 的核心 API 网关能力，并在真实服务器部署过程中补齐
运维层、访问层、出站网络层和 WebUI 可用性上的问题。当前版本以已验证过的运行形态为基准，目标是保留一套更适合自托管生产使用的最佳版本。

> 本项目仅供学习、研究和自托管部署验证。使用者应自行遵守相关服务条款、
> 平台规则和所在地法律法规。本项目不是 xAI、Grok、OpenAI 或任何相关平台的
> 官方项目。

## 项目定位

上游 `grok2api` 解决的是核心 API 转换和模型调用问题；本仓库关注的是把它
稳定地跑在自己的服务器上，并能长期维护。

自托管环境里，真正容易出问题的部分往往不是单次启动，而是这些细节：

- 出站链路需要在直连、WARP/Privoxy、Mihomo 节点之间切换。
- Admin、WebUI、节点控制面板不应该默认裸露在公网。
- 代理节点、账号库、token、真实域名和运行日志不能进入公开仓库。
- WebUI 功能需要适配实际账号池能力，而不是只按理想账号等级工作。
- 静态资源不能依赖可能失败的第三方 CDN。
- 生产环境补丁需要沉淀到源码，而不是长期依赖手工 override。

因此，`grok2api-enhanced` 不是简单改名版，而是在上游基础上做了面向部署和
运维的增强二开。

## 我的主要改进

| 方向 | 改进内容 |
| --- | --- |
| WebUI Masonry 生图 | Speed 模式在普通账号池下自动使用 `grok-imagine-image-lite` fallback，避免因缺少高级图像账号导致整批失败 |
| 生图并发与容错 | Lite 生图按槽位并发执行，单张失败只标记单个格子，不再触发整批 `TaskGroup` 异常 |
| 前端失败状态 | Masonry 前端新增 `slot_error` 状态，把网络/上游单槽失败与审核过滤区分显示 |
| Quality 能力边界 | Quality 模式在缺少 super/heavy 图像账号时给出明确提示，引导使用 Speed |
| 静态资源可靠性 | 移除 Masonry 页面外部 jsDelivr 字体样式依赖，避免 stylesheet 加载失败影响页面 |
| 出站代理管理 | Admin 配置页提供固定代理选项，支持 `privoxy` 与 `mihomo` 内部服务地址 |
| Mihomo 可视化 | 增加 `/mihomo/` 面板入口，用于查看节点、切换节点和测试延迟 |
| 私有访问层 | 提供 `access-gate` 与 `nginx-private.example.conf`，用于保护 Admin、WebUI 和节点面板 |
| Compose 组合部署 | 提供 WARP、Mihomo、Tunnel、Private Access 等可组合 overlay |
| 开源脱敏 | 提供示例配置和发布检查文档，避免真实账号、节点、域名、token 被提交 |

## 当前最佳运行形态

当前版本按 VPS 实测可用形态沉淀，推荐的完整部署组合是：

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.warp.yml \
  -f docker-compose.mihomo.yml \
  -f docker-compose.private.yml \
  -f docker-compose.tunnel.yml \
  up -d
```

这套形态包含：

- `grok2api`：核心 API/WebUI 服务，默认使用 `ghcr.io/s1oopx/grok2api-enhanced:latest` 增强镜像。
- `warp-proxy` 与 `privoxy`：提供稳定 WARP/Privoxy 出站路径。
- `mihomo`：提供可切换节点出站路径。
- `access-gate`：对公网入口做访问控制和反向代理。
- `cloudflared`：可选 Tunnel 入口。

推荐把 Admin、WebUI、Mihomo 面板放在私有访问层后面，只向可信来源开放。

## WebUI Masonry 生图行为

本版本重点修复了 Masonry 生图在普通账号池下不可用的问题。

### Speed 模式

Speed 模式优先尝试常规图像模型账号池。如果当前部署没有可用的常规图像账号，
会自动 fallback 到：

```text
grok-imagine-image-lite
```

这个路径使用 fast/basic 账号池，适合当前自托管环境中更常见的账号能力。

本版本对 lite fallback 做了并发槽位处理：

- 每轮最多 6 张图。
- 6 个槽位并发请求。
- 单个槽位失败时返回 `slot_error`。
- 已成功的图片继续显示。
- 不再因为一个槽位失败导致整批生成失败。

实测行为示例：

```text
Speed 6 张：5 张成功 + 1 个 slot_error
总耗时：约 11 秒
全局错误：0
```

### Quality 模式

Quality 模式仍然保留高级图像账号能力边界。当前账号池没有 super/heavy 图像
权限时，会返回明确提示：

```text
Quality mode requires super/heavy image accounts. Switch to Speed for the current basic pool.
```

这比原来的“生成失败”更清楚，也避免误判为服务异常。

### 前端状态区分

Masonry 前端现在区分三类结果：

| 状态 | 含义 |
| --- | --- |
| 成功 | 返回图片 URL，可打开图片 |
| 已过滤 | 上游审核或内容过滤 |
| 请求失败 | 网络、上游返回空结果或单槽生成失败 |

这样可以看清楚到底是内容被过滤，还是某个请求失败。

## 网络与出站设计

本项目把出站能力拆成稳定的内部代理端点，而不是把所有逻辑写死到主服务里。

```text
external client
  -> optional tunnel or reverse proxy
  -> access-gate
     -> grok2api:8000
     -> /mihomo/
     -> /mihomo-api/ -> mihomo:9090

grok2api
  -> selected internal proxy endpoint
     -> direct
     -> http://privoxy:8118
     -> http://mihomo:7890
```

常用出站选择：

| 出站方式 | Proxy URL | 适合场景 |
| --- | --- | --- |
| 直连 | 留空或关闭代理 | 服务器出口质量稳定，目标服务可直接访问 |
| WARP/Privoxy | `http://privoxy:8118` | 需要稳定、统一的 WARP 出口 |
| Mihomo | `http://mihomo:7890` | 需要手动选择节点或切换代理线路 |

对于账号绑定、会话绑定或风控敏感的工作负载，不建议在请求过程中频繁切换出口。
更推荐在批次之间切换节点。

## 快速开始

准备本地配置：

```bash
cp .env.example .env
cp mihomo/config.example.yaml mihomo/config.yaml
cp nginx-private.example.conf nginx-private.conf
```

如果使用 Tunnel overlay，将本地 Tunnel token 放到：

```text
./cloudflared.token
```

启动完整增强版：

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.warp.yml \
  -f docker-compose.mihomo.yml \
  -f docker-compose.private.yml \
  -f docker-compose.tunnel.yml \
  up -d
```

不使用 Tunnel 时，可以去掉 `docker-compose.tunnel.yml`，并按自己的入口层配置
反向代理。

查看服务：

```bash
docker compose ps
```

查看主服务日志：

```bash
docker logs -f grok2api
```

## 关键配置

### Admin 出站代理

进入 Admin 配置页后，建议按当前出口选择填写：

```text
egress mode: single_proxy
proxy_url: http://privoxy:8118 或 http://mihomo:7890
resource_proxy_url: http://privoxy:8118 或 http://mihomo:7890
```

通常建议 `proxy_url` 和 `resource_proxy_url` 使用同一个出口，避免 API 请求与
资源下载走不同线路导致排障困难。

### Mihomo 面板

Mihomo 面板入口：

```text
/mihomo/
```

Mihomo 控制 API 由私有访问层代理：

```text
/mihomo-api/
```

请将 `mihomo/config.yaml` 和 `nginx-private.conf` 中的示例 secret 替换为同一个
强随机值。

### 私有访问

`nginx-private.example.conf` 只作为模板，不应直接包含真实域名、真实 IP 白名单或
生产 secret 后提交。

建议保护这些路径：

- `/admin/`
- `/webui/`
- `/mihomo/`
- `/mihomo-api/`

## Compose 文件说明

| 文件 | 作用 |
| --- | --- |
| `docker-compose.yml` | 基础服务定义 |
| `docker-compose.warp.yml` | WARP/Privoxy 出站链路 |
| `docker-compose.mihomo.yml` | Mihomo 服务与节点面板 |
| `docker-compose.private.yml` | 私有访问网关 |
| `docker-compose.tunnel.yml` | Cloudflare Tunnel 入口 |

这些文件可以按环境组合。生产环境推荐显式写出所有 `-f` 文件，避免误用默认
Compose 文件导致服务缺失。

## 发布与脱敏边界

本仓库只保存可复用源码、模板和脱敏示例，不保存真实运行资产。

不要提交：

- `.env`
- `data/`
- `logs/`
- `cloudflared.token`
- `nginx-private.conf`
- `mihomo/config.yaml`
- 账号数据库
- 真实域名、真实 IP、真实节点
- Cookie、token、API key、订阅链接
- 任何运行日志或用户请求内容

发布前建议检查：

```bash
git status --short
git add -n .
git diff --cached --name-only
```

并参考：

- [Open Source Sanitization](docs/open-source-sanitization.md)
- [Private Access](docs/private-access.md)
- [Egress Strategy](docs/egress.md)
- [Architecture](docs/architecture.md)

## 适合谁使用

本项目适合：

- 想自托管 `grok2api`，并长期维护服务的人。
- 需要 WARP、Mihomo、多出口切换的人。
- 需要把 Admin、WebUI、代理面板放到私有访问层后面的人。
- 需要在普通账号池下使用 WebUI Masonry Speed 生图的人。
- 希望公开仓库和真实部署状态严格分离的人。

如果你只需要最原始的 API 网关能力，可以优先阅读上游项目：

- [jiujiu532/grok2api](https://github.com/jiujiu532/grok2api)

如果你需要的是可维护的自托管增强版，可以使用本仓库。

## 与上游的关系

本仓库是基于
[jiujiu532/grok2api](https://github.com/jiujiu532/grok2api)
的二次开发增强版。

保留上游项目能力、许可证和归属说明。本仓库新增内容主要集中在：

- 自托管部署组合
- 私有访问层
- 出站代理和节点管理
- WebUI Masonry 生图兼容性
- 生产环境可维护性
- 开源脱敏流程

## License

MIT。重新分发时请保留原项目许可证与归属信息。
