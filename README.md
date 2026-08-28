# grok2api-enhanced

`grok2api-enhanced` 是基于 [jiujiu532/grok2api](https://github.com/jiujiu532/grok2api) 的二次开发增强版，面向自托管、长期运行和私有化部署场景。

本项目保留上游 `grok2api` 的核心 API 网关能力，并在真实服务器部署过程中补齐运维层、访问层、出站网络层和 WebUI 可用性上的问题。

> 本项目仅供学习、研究和自托管部署验证。使用者应自行遵守相关服务条款、平台规则和所在地法律法规。本项目不是 xAI、Grok、OpenAI 或任何相关平台的官方项目。

---

## 快速开始 (Quickstart)

```bash
git clone https://github.com/s1oopX/grok2api-enhanced.git && cd grok2api-enhanced
cp config.example.json config.json  # 填入你的上游凭证与端口
docker compose -f docker-compose.yml -f docker-compose.mihomo.yml up -d
```

- API 网关监听：`http://localhost:8000/v1`
- 节点切换与看板：`http://localhost:8000/mihomo/`
- 管理控制台：`http://localhost:8000/admin`

---

## 项目定位

上游 `grok2api` 解决的是核心 API 转换和模型调用问题；本仓库关注的是把它稳定地跑在自己的服务器上，并能长期维护。

自托管环境里，真正容易出问题的部分往往不是单次启动，而是这些细节：

- 出站链路需要在直连、WARP/Privoxy、Mihomo 节点之间切换。
- Admin、WebUI、节点控制面板不应该默认裸露在公网。
- 代理节点、账号库、token、真实域名和运行日志不能进入公开仓库。
- WebUI 功能需要适配实际账号池能力，而不是只按理想账号等级工作。
- 静态资源不能依赖可能失败的第三方 CDN。
- 生产环境补丁需要沉淀到源码，而不是长期依赖手工 override。

---

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

---

## 推荐完整部署形态 (VPS 组合)

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
- `grok2api`：核心 API/WebUI 服务（增强镜像）。
- `warp-proxy` 与 `privoxy`：提供稳定 WARP/Privoxy 出站路径。
- `mihomo`：提供可切换节点出站路径。
- `access-gate`：对公网入口做访问控制和反向代理。
- `cloudflared`：可选 Cloudflare Tunnel 零暴露端口入口。

---

## WebUI Masonry 生图行为

本版本重点修复了 Masonry 生图在普通账号池下不可用的问题。

### Speed 模式
Speed 模式优先尝试常规图像模型账号池。如果当前部署没有可用的常规图像账号，会自动 fallback 到 `grok-imagine-image-lite`，并进行槽位并发：
- 每轮最多 6 张图，6 个槽位并发请求。
- 单槽超时默认 60 秒。
- 单张图失败只标记当前格子 `slot_error`，其他已成功图片正常展示并可放大、下载。

### Quality 模式
Quality 模式固定请求高阶生图链路。如果当前账号池没有绑定对应能力的账号，前端会明确提示，引导切回 Speed 模式。

---

## 安全与隐私建议

1. 不要将未加鉴权的 `8000` 端口直接暴露在公网。
2. 建议使用 `docker-compose.private.yml` 启用访问网关。
3. 生产环境中，日志文件和临时凭证请通过数据卷挂载并加入 `.gitignore`。

---

## 许可证

本项目基于原始上游协议开源，二次开发增强部分采用 [MIT License](LICENSE)。
