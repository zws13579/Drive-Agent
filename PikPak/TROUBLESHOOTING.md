# PikPak Agent 快速排障与错误代码速查手册 (TROUBLESHOOTING.md)

本手册专为 **PikPak 网盘管理 Agent** 编写。当 Agent 执行 API 请求、文件操作、云下载任务或 MCP 调用发生异常与报错时，应立即对照本手册进行原因判断与故障自愈处理。

---

## 快速速查表：HTTP 状态码与错误对照

| HTTP 状态码 / 错误码 | 典型报错信息 | 核心原因 | Agent 推荐处理策略 |
| :--- | :--- | :--- | :--- |
| **`401 Unauthorized`** | `invalid or missing token` / `unauthenticated` | Token 缺失、已过期（PAT 最长 1 年）或在后台被重新生成/删除 | 提示用户检查并更新 `.env` 中的 `PIKPAK_TOKEN`，严禁重试 |
| **`403 Forbidden`** | `PermissionDenied` / `scope insufficient` | 令牌缺少操作所需的权限范围（Scope） | 检查 PAT 权限；若需永久删除/清空回收站，注意官方限制 PAT 无法获得该权限 |
| **`403 Forbidden`** | `browser_signature_banned` / `Error 1010` | Cloudflare 反爬虫防护拦截（User-Agent 缺失或被封禁） | 请求头补齐标准浏览器 `User-Agent` |
| **`429 Too Many Requests`** | `rate limit exceeded` / 接口限频 | 调用频率过高触发服务端保护机制 | 指数退避重试（退避 2s, 4s, 8s），降低遍历速率 |
| **`400 / 409 Conflict`** | `quota_exceeded` / `transfer limit` | 达到已连接应用（Connected Apps）的 25% 配额上限 | 停止新建云下载与上传，检查下行是否降速至 100KB/s 并向用户汇报 |
| **`400 Bad Request`** | `invalid_task` / `task failed` | 磁力链接格式非法、种子文件损坏或触发违规拦截 | 检查 Magnet 链接规范性（`xt=urn:btih:`），提示用户链接可能失效或违规 |
| **`400 Bad Request`** | `file count exceeds limit` | 单个 BT/Torrent 任务包含文件数超过 **7000 个** | 告知用户官方限制单个种子最多 7000 个文件，无法分卷勾选 |
| **`404 Not Found`** | `file_not_found` / `task_not_found` | 目标文件 ID 不存在或已被移入回收站/彻底删除 | 重新调用文件列表接口 `drive/v1/files` 校验文件 ID 是否陈旧 |

---

## 1. 认证与权限异常（Auth Failures）

### 1.1 `401 Unauthorized`（凭证失效）
- **现象**：所有 API 返回 `{"error":10002, "error_key":"unauthorized"}`。
- **故障排查**：
  1. 检查根目录下的 `.env` 文件中是否存在 `PIKPAK_TOKEN`，且未被注释。
  2. 检查请求头是否包含 `Authorization: Bearer <TOKEN>`（注意 Bearer 与 Token 之间的空格）。
  3. **生命周期限制**：PikPak PAT 有效期最长为 1 年（默认 90 天，不支持永久令牌）。若用户在后台点击了“重新生成”，旧令牌会即刻失效。
- **Agent 行动指南**：
  - 明确提醒用户：“PikPak 访问令牌已失效或过期，请前往 PikPak 网页端【账号与安全 > 已连接的应用 > 个人访问令牌】重新生成，并填入 `.env` 文件。”
  - **严禁**在日志或报错信息中打印 Token 字符串。

### 1.2 `403 Forbidden`（权限不足）
- **现象**：读取正常，但在上传、移动、重命名或添加云任务时报错。
- **权限范围（Scopes）对照**：
  - 读取与下载：`files:read`
  - 上传、新建文件夹、重命名、移动：`files:write`
  - 移入回收站、从回收站恢复：`files:manage`
  - 离线下载（云添加）：`cloud_download`
  - **注意**：PikPak 官方规定 **PAT 令牌无法获得“永久删除”与“邀请”权限**。Agent 若需要彻底删除文件，应提醒用户无法通过 PAT 彻底删除，只能移入回收站。

---

## 2. 传输配额与速率限制（Quota & Rate Limits）

### 2.1 已连接应用 25% 配额熔断机制
PikPak 对第三方集成、脚本、API 调用采用**已连接应用传输配额保护规则**：
- **核心限制**：所有已连接应用（包括本 Agent 的 API 请求、WebDAV、MCP 等）**共同使用**当月有效总配额的 **25%**。
  - 云下载配额（月基础 40TB 的 25% = 10TB）
  - 下行配额（月基础 4TB 的 25% = 1TB）
  - 上传配额（月基础 1TB 的 25% = 250GB）
- **触发超限时的表现**：
  1. **下行流量**：不会直接中断，但会被强制限速至约 **100 KB/s**（下载和取链播放极慢）。
  2. **上传请求**：后续所有上传 API 直接被服务端拒绝。
  3. **云下载任务**：无法新建离线下载任务，已在运行的任务会继续跑完。
  4. **免费账号每日上限**：免费账号每日仅有 5GB 的已连接应用下行限额（新加坡时间 0:00 重置）。
- **Agent 行动指南**：
  - 调用 `https://api-drive.mypikpak.com/drive/v1/about` 监控容量与配额。
  - 若任务报错显示配额不足，向用户汇报当前已用配额与超限维度，避免盲目重试浪费流量。

---

## 3. 离线云下载任务异常（Cloud Download Failures）

### 3.1 极端有害内容合规拦截
- **现象**：添加磁力链接后，任务状态直接标记为失败、屏蔽或拒绝存入。
- **原因**：命中系统极端有害内容过滤规则（公认违法及严重违规资源）。
- **Agent 行动指南**：
  - 告知用户：“该资源被云盘系统安全策略拦截，无法离线保存。如确属正常资源误判，需在 PikPak 官方客户端内通过申诉功能反馈。”

### 3.2 任务文件数量超限
- **现象**：解析种子或磁力链接失败。
- **原因**：PikPak 目前单个 BT 任务最多支持 **7000 个文件**；且**不支持选择子文件进行部分下载**。
- **Agent 行动指南**：
  - 若解析出多于 7000 个文件，提示用户种子文件过多超限；无法单选子文件下载。

---

## 4. 文件与目录操作异常（File & Directory Operations）

### 4.1 根目录 ID 规范
- **规范**：PikPak API 中根目录 ID 是**空字符串 `""`**，而不是 `"/"` 或 `"0"`。
- **错误示范**：`GET /drive/v1/files?parent_id=/` 或 `parent_id=root` 会导致返回空列表或参数错误。

### 4.2 文件删除与存储空间未释放
- **现象**：调用删除接口将文件删除后，查询 `drive/v1/about` 发现已用存储空间没有减少。
- **原因**：删除操作默认只是将文件放入**回收站（Trash）**，依然计入已用存储容量。
- **Agent 行动指南**：
  - 向用户说明文件当前已移入回收站，空间仍被占用；如需释放空间需在回收站清空（受限于 PAT 权限，可能需用户在网页端手动清空回收站）。

### 4.3 分页遍历死循环与漏文件
- **规范**：调用 `/drive/v1/files` 列目录时，一页请求仅返回当前批次：
  - 必须检查响应中的 `next_page_token`。
  - 当且仅当 `next_page_token` 为非空字符串时，将其作为 query 参数 `page_token` 发起下一次请求，直至为空。
  - 请求参数 `-l`（limit）最低有效值为 10，低于 10 会被服务端强制视为 10。

---

## 5. 网络请求与 Cloudflare 防护异常

### 5.1 Cloudflare 403 拦截（Error 1010）
- **现象**：HTTP 请求直接返回 HTML 或 `{"error_code": 1010, "error_name": "browser_signature_banned"}`。
- **原因**：使用默认的 `curl/7.x` 或 Python `urllib` 请求头被 Cloudflare 防火墙判定为爬虫工具。
- **解决标准代码**：
  - 在所有请求头中明确加入标准浏览器 User-Agent：
    ```python
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
        "Authorization": f"Bearer {token}",
    }
    ```
