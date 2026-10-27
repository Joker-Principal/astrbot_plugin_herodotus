# astrbot_plugin_herodotus

> Herodotus 系统的 **AstrBot 客户端插件**（Python / Star）。
>
> 这是一个**瘦客户端**：只做「捕获转发」与「召回注入」两件事，不含任何记忆逻辑。AstrBot 侧的插件存储是**插件私有**的（KV 与 `data/plugin_data/` 均按插件隔离），**无法跨系统共享**——这正是记忆必须放在独立服务 `clio` 里的原因。

## 职责

| 接缝 | 作用 |
| --- | --- |
| `@filter.event_message_type(filter.EventMessageType.ALL)` | 捕获消息事件 → `POST /ingest` |
| `@filter.on_llm_request()` | 取召回结果 → `request.extra_user_content_parts.append(TextPart(...))` + `.mark_as_temp()` |
| `context.add_llm_tools(...)` 或 `@filter.llm_tool` | 注册记忆工具（或经 MCP 提供） |
| `pages/`（插件 Pages） | WebUI 记忆面板与设置页 |
| `skills/` | 随插件分发的 Skill（可选） |

## 规划中的目录布局

```
astrbot_plugin_herodotus\
├─ metadata.yaml        （P8）插件元数据；AstrBot 依赖它识别插件
├─ main.py              （P8）Star 子类入口
├─ requirements.txt     （P8）插件依赖（AstrBot 用 pip 管理）
├─ pages\               WebUI 页面
├─ skills\              随插件 Skills
└─ tests\
```

> AstrBot 规范：插件名推荐以 `astrbot_plugin_` 开头、全小写、不含空格；**持久化数据必须放在 `data/` 目录下而非插件自身目录**（否则更新 / 重装会被覆盖）。本插件的持久化只用到配置，记忆一律在服务端。

## 契约落点

契约正文在 [`../docs/PROTOCOL.md`](../docs/PROTOCOL.md) §5（C1–C7）。本文件**不复述**，只记在这个框架里怎么落。

| 契约 | 本框架的落点 |
| --- | --- |
| C1 降级不阻塞 | 捕获与召回异常一律吞掉走 no-op；首次失败只告警一次 |
| C2 动态内容只进消息侧 | `@filter.on_llm_request()` 里 `request.extra_user_content_parts.append(TextPart(...))`；**不碰 `system_prompt`** |
| C3 稳定内容进系统提示词 | 仅人格定义与 L3 画像快照 |
| C4 / C5 数据非指令 + 防回捕 | 🟡 待定：`.mark_as_temp()`（AstrBot ≥ v4.24.0）使注入内容不进会话历史 |
| C6 显式 `personaId` | 由插件配置映射而来，逐请求携带 |
| C7 版本校验 | 启动时 `GET /health` 比对协议版本，不匹配则整体禁用并报错 |

> C2 的依据：AstrBot 官方文档明确警告，把每轮变化的内容追加到 `system_prompt` 会破坏模型服务端的提示词缓存，成本增加约 7–20 倍。

### 框架自己的约束（与契约无关）

- 异步网络请求用 `aiohttp` / `httpx`，不用 `requests`（AstrBot 开发规范）。
- 插件持久化只放配置；记忆一律在服务端。

## 依赖与环境

- `uv` 或 `npm`（MCP 服务器用它们启动）——若通过 MCP 挂载 `clio`，需在 AstrBot WebUI 的 `插件 → MCP` 中配置。
- MCP 支持需 **AstrBot ≥ v3.5.0**。

## 许可

**GPL-3.0**。归属声明见主仓库 [`NOTICE`](../NOTICE)。

AstrBot 侧的注入接缝依据官方文档实现：
<https://docs.astrbot.app/dev/star/guides/ai.html>
