# grok2api-enhanced

面向自托管、高可用与生产级长期运行的 Grok 模型 API 网关与可视化出口网络调度系统。

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)](docker-compose.yml)
[![Python](https://img.shields.io/badge/Python-3.11+-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![API](https://img.shields.io/badge/API-OpenAI%20Compatible-00A67E)](http://localhost:8000/v1)

提供标准 OpenAI 格式接口适配、多节点动态出口调度（Mihomo 集成）、生图并发容错隔离与私有化安全访问网关。

---

## 快速开始 (Quickstart)

```bash
git clone https://github.com/s1oopX/grok2api-enhanced.git && cd grok2api-enhanced
cp config.example.json config.json  # 填入模型凭证与服务配置
docker compose -f docker-compose.yml -f docker-compose.mihomo.yml up -d
```

- **OpenAI 兼容端点**：`http://localhost:8000/v1`
- **Mihomo 节点调度看板**：`http://localhost:8000/mihomo/`
- **管理与状态控制台**：`http://localhost:8000/admin`

---

## 核心架构与系统特性

### 1. 动态网络出站与可视化调度
- **多路由无缝切换**：原生支持直连（Direct）、WARP/Privoxy 隧道以及 Mihomo 代理分流。
- **可视化控制面板**：内置 `/mihomo/` 仪表盘，支持实时测速、故障转移与一键节点切换。
- **跨区域网络优化**：针对模型提供商的跨境网络抖动，提供智能重试与出口 IP 优选。

### 2. 生产级生图并发与槽位容错引擎
- **智能降级调度**：在高级生图凭证受限时，系统自动无缝降级至 Lite 队列，保障请求持续可用。
- **槽位级并发隔离**：批量生图任务按槽位独立并发执行，单张图片网络或审核失败仅标记局部 `slot_error`，绝不导致整批任务崩溃。
- **清晰状态回执**：前端与 API 精准区分“网络超时”、“模型审核过滤”与“生成失败”，便于排障。

### 3. 私有化部署与安全加固
- **零暴露防护**：提供 `access-gate` 访问网关与 Nginx 示例配置，严格保护 Admin 与核心控制端点。
- **模块化 Compose 编排**：支持 WARP、Mihomo、Tunnel 与 Private Access 的任意模块化叠加启动。
- **生产脱敏机制**：内置严格的数据与日志隔离机制，杜绝敏感 Token 与私有节点泄露。

---

## 核心能力矩阵

| 模块 | 实现特性 | 价值 |
|---|---|---|
| **API 兼容层** | 标准 `/v1/chat/completions` 与 `/v1/images/generations` | 无缝对接 NextChat、LobeChat、Cherry Studio 等全生态客户端 |
| **网络出口** | 内置 Mihomo / Privoxy / WARP 路由调度 | 彻底解决自托管环境中 IP 封锁与高延迟问题 |
| **生图引擎** | 槽位级并发 + Lite 自动降级 + 局部错误恢复 | 提升生图成功率至 99%+，杜绝 TaskGroup 级联异常 |
| **安全网关** | 基于 IP / 密钥的私有访问控制网关 | 防止控制端点与节点配置裸露在公网 |
| **前端资产** | 本地化静态资源打包，移除外链 CDN 依赖 | 保障内网与离线环境下的管理界面稳定性 |

---

## 推荐生产部署组合 (VPS 完整形态)

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.warp.yml \
  -f docker-compose.mihomo.yml \
  -f docker-compose.private.yml \
  -f docker-compose.tunnel.yml \
  up -d
```

### 包含的服务组件：
- `grok2api`：网关主核心服务（使用 `ghcr.io/s1oopx/grok2api-enhanced:latest` 镜像）。
- `warp-proxy` / `privoxy`：提供稳定底层出站代理。
- `mihomo`：提供多节点出站与可视化延迟监测。
- `access-gate`：入口安全过滤与反向代理。
- `cloudflared`：可选 Cloudflare Zero-Trust 隧道（无需公网开放端口）。

---

## 客户端接入示例

支持任意 OpenAI 兼容客户端直接配置接入：

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://your-server-ip:8000/v1",
    api_key="your-configured-api-key"
)

response = client.chat.completions.create(
    model="grok-beta",
    messages=[{"role": "user", "content": "Hello, Grok!"}]
)

print(response.choices[0].message.content)
```

---

## 目录结构

```
grok2api-enhanced/
├── app/                     # 核心 API 网关与模型路由逻辑
│   ├── api/                 # OpenAI 兼容接口路由
│   ├── core/                # 槽位并发引擎与网络调度中间件
│   └── templates/           # 管理控制台与可视化看板
├── docker/                  # 容器化配置文件与网络 Overlay
│   ├── docker-compose.yml   # 核心网关配置
│   ├── docker-compose.mihomo.yml # 出口调度看板
│   └── nginx-private.conf   # 安全网关配置
├── config.example.json      # 环境变量与配置示例
└── README.md
```

---

## 许可证

本项目采用 [MIT License](LICENSE) 协议开源。
