# grok2api-enhanced

面向自托管场景的 grok2api 部署增强版。

这个仓库的重点不是改写上游项目的产品说明，而是把一套更容易维护的
Docker 部署形态整理出来：出站代理可切换、Mihomo 节点可视化、管理入口可
私有化，并给出脱敏后的示例配置。

> 本项目仅供学习、研究和自托管部署验证。请遵守相关服务条款、平台规则和所在地法律法规。

## 这个版本解决什么问题

自托管 API 网关通常不是“能跑起来”就结束了。长期运行时还会遇到几个更实际
的问题：

- 出站链路需要在直连、WARP/Privoxy、Mihomo 节点之间切换。
- 代理配置不应该只藏在配置文件里，管理页面需要能直接选择常用出口。
- 节点状态、延迟测试和当前选择需要有一个可视化入口。
- Admin、节点面板等敏感页面不应该默认裸露在公网。
- 公开仓库不能包含真实域名、账号库、代理凭证、token 或运行日志。

`grok2api-enhanced` 围绕这些运维问题做增强，尽量把真实部署和开源代码分离。

## 增强内容

| 模块 | 内容 |
| --- | --- |
| Admin 配置页 | 出站代理 URL 增加固定选择项：`http://privoxy:8118` 与 `http://mihomo:7890` |
| Mihomo 面板 | 增加 `/mihomo/` 入口，用于查看当前节点、切换节点和测试延迟 |
| Compose Overlay | 提供 `docker-compose.mihomo.yml`，可叠加到现有部署 |
| 私有访问示例 | 提供 `nginx-private.example.conf`，用于 Admin 与面板入口的访问控制 |
| 示例配置 | 提供 `mihomo/config.example.yaml`，不包含任何真实节点或订阅 |
| 发布安全 | 提供 `docs/open-source-sanitization.md`，用于发布前检查敏感信息 |

## 部署形态

这个仓库保留上游 grok2api 的核心服务，并增加几组可组合的部署层。

```text
client
  -> optional private access gate
  -> grok2api
     -> optional companion network helpers
     -> selected egress

egress options
  direct
  http://privoxy:8118
  http://mihomo:7890
```

推荐按实际环境选择一个稳定出口长期使用。对于绑定账号、会话或风控状态的
工作负载，不建议频繁随机轮换节点。

## 快速开始

准备示例配置：

```bash
cp .env.example .env
cp mihomo/config.example.yaml mihomo/config.yaml
cp nginx-private.example.conf nginx-private.conf
```

启动基础服务加增强层：

```bash
docker compose \
  -f docker-compose.warp.yml \
  -f docker-compose.private.yml \
  -f docker-compose.mihomo.yml \
  up -d
```

只启用 Mihomo 增强层：

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.mihomo.yml \
  up -d
```

查看服务：

```bash
docker compose ps
```

## 出站代理选择

进入 Admin 配置页后，找到网络代理相关配置：

```text
egress mode: single_proxy
proxy_url: http://privoxy:8118 或 http://mihomo:7890
resource_proxy_url: http://privoxy:8118 或 http://mihomo:7890
```

常见选择：

| 选择 | 适合场景 |
| --- | --- |
| `direct` | 服务器出口质量稳定，目标服务可直接访问 |
| `http://privoxy:8118` | 需要走 WARP/Privoxy 这一固定链路 |
| `http://mihomo:7890` | 需要使用 Mihomo 中选定的节点 |

如果启用了 Mihomo，面板入口为：

```text
/mihomo/
```

建议把 Admin 和 `/mihomo/` 放在私有访问层后面，只允许可信来源访问。

## 文档

- [Egress Strategy](docs/egress.md)
- [Private Access](docs/private-access.md)
- [Open Source Sanitization](docs/open-source-sanitization.md)
- [Enhanced Variant Notes](README.enhanced.md)

## 发布前检查

公开仓库不应该包含下面这些内容：

- `.env`
- `data/`
- `logs/`
- `cloudflared.token`
- `nginx-private.conf`
- `mihomo/config.yaml`
- 账号数据库
- 真实域名、真实 IP、真实代理节点
- Cookie、token、API key、订阅链接

发布前建议执行：

```bash
git status --short
git add -n .
git diff --cached --name-only
```

并按 `docs/open-source-sanitization.md` 做关键词扫描。

## 与上游的关系

本仓库是基于
[jiujiu532/grok2api](https://github.com/jiujiu532/grok2api)
整理的部署增强版本，保留原项目能力、许可证和必要归属说明。

本仓库新增内容主要集中在：

- 出站代理选择体验
- Mihomo 可视化入口
- Docker Compose 部署组合
- 私有访问示例
- 公开发布脱敏流程

如果你只需要原始 API 网关能力，可以优先阅读上游项目；如果你更关心自托管
后的出口管理和运维边界，可以参考本仓库的增强配置。

## License

MIT. 重新分发时请保留原项目许可证与归属信息。
