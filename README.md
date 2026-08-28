# AstrBot Artalk Moderation

接收 Artalk 新评论 Webhook，将待审评论放入队列，交由 AstrBot 已配置的 LLM 审核，并把结果同步回 Artalk。

## 功能

- 接收 Artalk 内网 Webhook；请求成功入队即返回 HTTP `200`，不阻塞 Artalk 发评流程
- 启动时可选扫描一次历史待审评论，并每 30 分钟或更久定时扫描
- 通过有界队列和可配置并发控制模型请求
- 四种审核结论和对应操作：通过（公开）、折叠（已通过但折叠保留）、拒绝（删除）、待审核（保持待审）
- LLM 请求失败、无法判断或返回格式不合规时，评论保持待审核，并向通知会话报告失败
- 仅缓存本次运行中已通知的待审核评论 ID，避免定时扫描重复通知；缓存有上限，重载插件后清空
- 审核通知和事件上下文写入指定 AstrBot 会话，供 Agent 继续处理
- 提供公开、折叠、删除三个受管理员限制的 Agent Tools
- 提供绑定通知会话、手动扫描和状态查询命令

## 依赖与网络

- 可运行插件 Web API 和 LLM Provider 的 AstrBot
- 一个 AstrBot 容器可访问的 Artalk HTTP API
- 一个具有 Artalk 管理员权限的邮箱账号，用于调用评论更新和删除接口

`artalk_url` 应填写 AstrBot 运行环境可访问的完整地址，例如：

```text
http://127.0.0.1:23366
https://comments.example.com
```

如果两者运行在同一 Docker 网络中，可使用服务名和容器端口，例如 `http://artalk:23366`。此时 `127.0.0.1` 指向 AstrBot 容器本身，而不是 Artalk 容器。

## 安装

在 AstrBot WebUI 插件市场安装，或将本目录放入 `AstrBot/data/plugins`，然后在 WebUI 重载插件。

插件没有额外 Python 依赖。

## 快速配置

1. 在插件配置中填写 `artalk_url`、`artalk_site_name`、`artalk_admin_email`、`artalk_admin_password` 和 `llm_provider_id`。
2. 把自己的平台 UID 写入 `admin_uids`。
3. 在要接收审核通知的会话发送 `/artalk_bind`。
4. 在 Artalk 配置中开启 `moderator.pending_default: true`，使新评论先进入待审状态。
5. 在 Artalk 的 `admin_notify.webhook` 中填写下方 Webhook 地址。
6. 重载插件后发送 `/artalk_status`；必要时发送 `/artalk_scan` 扫描现有待审评论。

### Artalk Webhook 配置

AstrBot Plugin Web API 的路径为：

```text
http(s)://<AstrBot可访问地址>:<端口>/api/v1/plugins/extensions/astrbot_plugin_artalk_moderation/webhook?api_key=你的AstrBot插件权限APIKey
```

`<AstrBot可访问地址>:<端口>` 必须从 Artalk 的运行环境可访问：同机部署可使用回环地址或本机网络地址，跨主机部署可使用受保护的域名/IP，Docker 同网络部署则可使用 `astrbot:6185`。`api_key` 应是具有插件权限的 AstrBot API Key；如果 Key 含有 URL 保留字符，必须先进行 URL 编码。不要使用包含未编码 `%`、`#`、`&` 等字符的 URL。

最小 Artalk 配置示例：

```yaml
moderator:
  pending_default: true

admin_notify:
  notify_pending: true
  webhook:
    enabled: true
    url: "http://<AstrBot可访问地址>:<端口>/api/v1/plugins/extensions/astrbot_plugin_artalk_moderation/webhook?api_key=替换为插件权限APIKey"
```

Webhook 只负责通知插件；插件会重新从 Artalk API 获取评论原文，不信任 Webhook Body 中的评论内容。

## 审核结果

| 模型首行 | 结论 | Artalk 操作 |
| --- | --- | --- |
| `通过` | 通过 | `is_pending=false`、`is_collapsed=false`，公开评论 |
| `折叠` | 折叠 | `is_pending=false`、`is_collapsed=true`，审核通过但折叠保留 |
| `拒绝` | 拒绝 | `DELETE /api/v2/comments/{id}`，删除评论 |
| `待审核` | 待审核 | `is_pending=true`、`is_collapsed=false`，保留人工处理 |

模型输出格式固定为两行纯文本：第一行必须是上表四项之一，第二行为不超过 30 字的原因。模型请求失败、首行不在这四项中或无法可靠判断时，插件不会公开、折叠或删除评论，而是保持待审核。

`allow_keyword`、`collapse_keyword`、`reject_keyword`、`review_keyword` 仅影响通知和 Agent 上下文中的显示文案；模型仍固定使用上表的中文首行，避免自定义词导致模型输出不稳定。

## 主要配置

| 配置项 | 默认值 | 说明 |
| --- | --- | --- |
| `artalk_url` | 空 | AstrBot 可访问的 Artalk 地址，例如 `http://127.0.0.1:23366` 或 `https://comments.example.com` |
| `artalk_site_name` | 空 | Artalk 站点名称，必须与前端 `site` 一致 |
| `artalk_admin_email` / `artalk_admin_password` | 空 | 用于登录 Artalk 管理 API |
| `llm_provider_id` | 空 | 审核使用的 AstrBot LLM Provider |
| `notify_umo` | 空 | 审核通知目标；推荐用 `/artalk_bind` 自动设置 |
| `admin_uids` | `[]` | 可执行命令和 Agent 评论工具的平台 UID |
| `max_concurrent_moderations` | `1` | 同时执行的 LLM 审核数 |
| `webhook_queue_size` | `100` | Webhook 审核队列容量；满时回调返回 503 |
| `llm_failure_cache_size` | `100` | 当前运行内、已通知待审核评论的去重缓存上限 |
| `scan_on_start` | `true` | 是否在插件启动后扫描已有待审评论 |
| `scan_interval` | `1800` | 定时扫描间隔，单位秒，最小 1800 秒 |
| `request_timeout` | `15` | Artalk HTTP 请求超时，单位秒 |
| `enable_agent_tools` | `true` | 是否注册评论管理 Agent Tools |
| `max_comment_length` | `3000` | 提交给模型的评论正文最大字符数 |
| `moderation_instruction` | 内置默认说明 | 审核标准；输出格式由代码固定控制 |

`notification_template`、`agent_context_template` 和 `failure_notification_template` 均可配置，且不能为空。支持的变量以配置页面提示为准。

## 命令

| 命令 | 作用 |
| --- | --- |
| `/artalk_bind` | 将当前会话绑定为审核通知目标 |
| `/artalk_scan` | 立即扫描 Artalk 当前站点的未折叠待审评论并入队 |
| `/artalk_status` | 查看队列、并发、扫描时间、错误和 LLM 失败缓存状态 |

以上命令都受 `admin_uids` 限制。

## Agent Tools

| Tool | 作用 |
| --- | --- |
| `artalk_approve_comment` | 公开一条评论 |
| `artalk_collapse_comment` | 审核通过后折叠保留一条评论 |
| `artalk_delete_comment` | 删除一条评论 |

三个 Tool 均要求调用者位于 `admin_uids`，并会校验评论属于已配置的 Artalk 站点。

## Artalk API 行为

插件使用 Artalk v2 HTTP API：`PUT /api/v2/comments/{id}` 更新公开、待审和折叠状态，`DELETE /api/v2/comments/{id}` 删除评论。Artalk 管理 API 的登录和评论接口均需要有效管理员身份，详情见 [Artalk HTTP API](https://artalk.js.org/http-api)。

## 安全说明

- 评论正文是外部不可信输入。插件将其与系统审核规则分离，并要求模型忽略评论中的指令。
- 最终仅接受固定的四种模型首行；任何异常输出均不会触发公开、折叠或删除。
- Artalk 管理员密码和 AstrBot API Key 均属于敏感信息；不要提交到仓库、日志或公开配置。
- 不要把 Artalk API 或 AstrBot WebUI 无保护地暴露到公网。容器间调用建议限制在可信 Docker 网络。
- 删除操作不可恢复；请按实际内容风险谨慎调整 `moderation_instruction`，必要时让高风险类别进入“待审核”或“折叠”。

## 发布前检查

```bash
python3 -m py_compile main.py
python3 -m json.tool _conf_schema.json >/dev/null
```

## 开发说明

本项目采用纯 **Vibe Coding** 方式开发，代码与文档由 AI 根据维护者提出的需求进行生成和编辑，并由维护者在实际 AstrBot 与 Artalk 环境中测试、验收和决定发布。使用或贡献代码时，请像审查其他软件一样独立检查其安全性、兼容性与行为。
