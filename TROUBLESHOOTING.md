# 通用型网盘管理快速排障与错误速查指南 (TROUBLESHOOTING.md)

本指南面向**通用型网盘管理 Agent**，提炼跨协议、跨平台（WebDAV、REST API、gRPC、S3、FUSE 虚拟挂载）的核心错误模式、底层根因与自动化自愈策略。去除了针对特定商业网盘或特定挂载软件的强耦合配置，专注于通用云存储治理中的典型故障排查。

---

## 快速速查表：通用错误状态码与故障定位

| 错误类别 | 通用状态码 / 标志报错 | 核心根因 | Agent 推荐自愈与处理策略 |
| :--- | :--- | :--- | :--- |
| **凭证失效** | HTTP `401 Unauthorized`<br>gRPC `UNAUTHENTICATED`<br>`InvalidToken / Expired` | API Token、Cookie、WebDAV 密码过期、缺失或被服务端吊销 | 立即中止同类请求，严禁死循环重试；提示用户检查并更新环境变量或本地 `.env` 凭证（严禁在日志中打印凭证） |
| **权限受限** | HTTP `403 Forbidden`<br>gRPC `PERMISSION_DENIED`<br>POSIX `EACCES / os error 13` | 令牌缺乏写入/删除权限；挂载卷属主不匹配；或存储驱动处于“只读”保护状态 | 检查当前操作范围（Scope）；检查挂载点目录属主与 UID/GID 权限；核实云端是否开启写保护 |
| **资源不存在** | HTTP `404 Not Found`<br>gRPC `NOT_FOUND`<br>POSIX `ENOENT` | 路径不存在；远程文件已被其他进程移动/删除；或本地索引缓存陈旧 | 重新调用目录遍历接口刷新本地缓存；验证路径父级目录是否存在 |
| **目标冲突** | HTTP `409 Conflict`<br>gRPC `ALREADY_EXISTS`<br>POSIX `EEXIST` | 目标路径已存在同名文件/文件夹；或并发重命名触发竞态条件 | 触发防覆盖自愈机制：自动按规范追加递增后缀 `(1)`，或跳过重复项 |
| **限频与风控** | HTTP `429 Too Many Requests`<br>gRPC `RESOURCE_EXHAUSTED`<br>`SlowDown / RateLimit` | 请求 QPS 超过服务商安全阈值；或短时间内发起过多并发连接 | 触发标准指数退避重试（2s, 4s, 8s + 随机抖动）；工作线程并发数下调至 2~3 个 |
| **容量/配额超限** | HTTP `507 Insufficient Storage`<br>`QuotaExceeded / DiskFull` | 云端个人空间已满；单日上行/下行流量配额耗尽 | 停止上传或云下载任务；向用户汇报当前已用配额与超限维度，避免继续消耗网络带宽 |
| **挂载点冲突** | POSIX `EBUSY`<br>`Mount point is not empty / Busy` | 挂载目标目录非空（存在隐藏文件或残留数据）；或目录被其他进程占用 | 检查挂载点目录残留文件，经确认后清理无用残留，保持挂载点绝对干净 |
| **服务不可达** | HTTP `502 / 503 / 504`<br>gRPC `UNAVAILABLE`<br>`Connection refused / Reset` | 存储后端进程未启动；反向代理缓冲导致流中断；网络临时中断 | 执行基础服务心跳探活；检查反向代理是否关闭 Buffering；实施间歇性健康重试 |
| **网络与握手** | `TLS handshake timeout`<br>`Network is unreachable / DNS error` | DNS 污染无法解析 API 域名；系统代理配置异常；网络防火墙拦截 | 探测网络连通性；验证 DNS 状态与代理链路；提示检查主机外网连通性 |

---

## 1. 认证鉴权与会话生命周期异常 (Authentication & Session)

### 1.1 凭证失效与静默失败
- **典型现象**：批量移动/重命名执行到一半突然报错 `401 Unauthorized` 或连续失败。
- **故障根因**：
  1. 长期运行的大型整理任务跨越了临时 Token 或 OAuth Access Token 的有效生命周期。
  2. 用户在网页端或其它客户端重新登录/重置了密码，导致原有 Session/Token 瞬间作废。
- **Agent 自愈与防范准则**：
  - **快速熔断**：一旦捕获到连续 2 次 401 错误，立即暂停整个批处理队列，防止触发云端防暴力破解封禁。
  - **安全审计**：向用户汇报“当前存储凭证已失效，请更新配置”，**严禁**在错误堆栈、控制台或向用户输出的信息中打印 Token 明文。

---

## 2. 虚拟文件系统与底层挂载异常 (FUSE & Virtual Filesystem)

### 2.1 挂载点非空冲突与占用 (Mount Point Conflicts)
- **典型现象**：执行挂载命令时系统报错 `mount point is not empty` 或 `device or resource busy`。
- **故障根因**：虚拟文件系统（FUSE）出于数据安全考量，要求挂载目标必须是一个**绝对空的本地目录**。若目录下存在操作系统生成的隐藏文件（如 `.DS_Store`、`Thumbs.db`）、历史崩溃残留文件或由上一个未正常卸载的僵尸进程占用，挂载将被拒绝。
- **Agent 标准排查流程**：
  ```bash
  # 1. 检查是否存在隐藏文件或目录残留
  ls -la /path/to/mount_point

  # 2. 检查是否有未退出的进程正在占用挂载点
  lsof +D /path/to/mount_point 2>/dev/null || fuser -v /path/to/mount_point 2>/dev/null

  # 3. 若为残留挂载，安全卸载；若是孤立文件，在确认无敏感数据后清理
  umount -l /path/to/mount_point 2>/dev/null || true
  ```

### 2.2 权限不匹配与只读挂载 (Read-Only & EACCES)
- **典型现象**：挂载成功且能读取文件列表，但在尝试 `mkdir`、`move` 或 `rename` 时返回 `Read-only file system` 或 `os error 13 (Permission denied)`。
- **故障根因**：
  1. 挂载时未声明权限继承（如 Linux 下普通用户运行缺少 `allow_other` 配置）。
  2. 容器环境下缺少宿主机内核交互特权（如未声明特权模式或 FUSE 驱动设备映射）。
  3. 云盘配置中以“只读模式（Read-Only）”挂载。

---

## 3. 频率风控、并发保护与配额熔断 (Rate Limits & Quota)

### 3.1 API 限频与防封禁机制 (Rate Limiting & Backoff)
- **典型现象**：执行全盘深度递归扫描或批量整理 500+ 文件时，突发 `429 Too Many Requests`，随后所有接口短时间内均无法响应。
- **故障根因**：短时间内高频向存储 API 发起并发请求（如批量获取元数据、并发频繁移动），触发了云存储平台的 QPS 阈值或防爬虫风控。
- **标准指数退避（Exponential Backoff with Jitter）实现**：
  ```python
  import time, random

  def safe_cloud_execute(action_fn, max_retries=5, base_delay=2.0):
      """通用的带抖动指数退避执行器"""
      for attempt in range(max_retries):
          try:
              return action_fn()
          except Exception as e:
              # 判断是否命中 429 限流或连接被重置
              is_rate_limit = "429" in str(e) or "RESOURCE_EXHAUSTED" in str(e)
              if not is_rate_limit or attempt == max_retries - 1:
                  raise
              # 延迟计算：base_delay * 2^attempt + jitter
              sleep_time = (base_delay * (2 ** attempt)) + random.uniform(0.1, 1.0)
              time.sleep(sleep_time)
  ```
- **并发控制基线**：
  - 针对通用网盘 API，单个存储账号的写操作并发 worker 建议严格控制在 **3 ~ 5** 个；严禁为了追求速度开启数十个并发线程。

### 3.2 存储空间与配额熔断
- **典型现象**：移动文件、离线下载或重命名失败，提示 `QuotaExceeded` 或 `507 Insufficient Storage`。
- **排查路径**：
  1. 区分“物理存储容量超限”与“单日传输流量配额耗尽”。
  2. **回收站占用陷阱**：在大多数云存储中，将文件“删除”往往只是移入“回收站/Trash”，已用空间**并不会减少**。若容量爆满，需提醒用户清空回收站才能真正释放配额。

---

## 4. 批量操作、路径规范与冲突处理 (File Operations & Integrity)

### 4.1 跨平台路径非法字符与超长路径
- **典型现象**：在云端重命名或跨目录移动时报 `400 Bad Request` 或 `Invalid Argument`。
- **通用规范约束**：
  - **禁止非法字符**：文件名中不得包含操作系统保留字符：`\ / : * ? " < > |`
  - **首尾空白与控制字符**：清理首尾空格、制表符及不可见控制字符；避免以英文句点 `.` 结尾。
  - **路径长度上限**：单文件名不超过 255 字节，完整路径不超过 1024 字节。

### 4.2 目标路径同名冲突与原子替换保护
- **原则**：严禁在未获用户显式授权的情况下执行静默覆盖。
- **冲突自愈规则**：
  - 在移动或重命名时，预先探测目标路径是否存在。
  - 若目标已存在同名文件：
    1. 比较两者的文件大小与 Hash（若支持）；如果完全一致，记录为重复项并跳过；
    2. 如果内容不同或无法获取 Hash，自动在文件名扩展名之前追加递增编号：
       `filename.ext` -> `filename (1).ext`。

### 4.3 目录修剪与误删防护
- **空目录修剪规则**：移动完文件后清理空文件夹时，必须确保该文件夹下**子文件数 == 0 且子目录数 == 0**。
- **媒体关键文件防护白名单**：
  - **严禁清理**：字幕文件（`.srt`, `.ass`, `.sub`, `.vtt`）、元数据（`.nfo`）、海报图片（`poster.jpg`, `fanart.jpg`、`cover.png`），无论体积大小（即便小于 1KB），均受白名单严格保护。

---

## 5. 网络连通性、代理与反向代理流控 (Network & Reverse Proxy)

### 5.1 反向代理缓冲导致的实时通信/长连接阻断
- **典型现象**：通过 Nginx / 反向代理访问管理接口时，网页端白屏、长时间无响应、长任务进度条卡死、或 SSE / gRPC / 双向流连接频繁断开。
- **故障根因**：多数反向代理服务器默认开启了 `proxy_buffering`（代理缓冲），会试图将服务端的流式响应全部缓存后再推送给客户端，从而彻底破坏了流式协议。
- **通用代理修正配置**：
  ```nginx
  # 必须关闭代理缓冲，确保 SSE、gRPC 与长轮询实时传输
  proxy_buffering off;
  proxy_request_buffering off;
  proxy_http_version 1.1;
  proxy_set_header Connection "";
  proxy_read_timeout 86400s;
  proxy_send_timeout 86400s;
  ```

### 5.2 网络握手超时与 DNS 污染
- **排查步骤**：
  1. 测试对云存储服务域名或 API Endpoint 的 DNS 解析与网络连通性。
  2. 若遇到 `TLS handshake timeout`，排查系统是否有透明代理、全局环境变量 `HTTP_PROXY` / `HTTPS_PROXY` 阻断了内网或局域网访问（必要时配置 `NO_PROXY=localhost,127.0.0.1,local`）。

---

## 6. 批处理故障隔离与一键回滚流程 (Rollback & Audit)

### 6.1 中途异常与故障隔离
- 当批量任务在处理到第 50 个文件时发生异常（如网络中断、网盘掉线）：
  1. **捕获中断**：捕获异常并立即停止后续任务派发，杜绝错误雪崩；
  2. **锁定清单**：立即将当前已经成功执行的 49 项操作以及失败的第 50 项操作完整写入 `manifests/manifest_<timestamp>.json`；
  3. **状态报告**：向用户汇报已成功数量、失败数量及失败原因，并提示是否发起逆向回滚。

### 6.2 自动化逆向回滚（Rollback）三步法则
1. **加载清单**：读取中断或需要撤销的 `manifest_<timestamp>.json`。
2. **反向映射**：将所有 `target_path` 作为操作源，`original_path` 作为操作目的。
3. **前置探测与逆向执行**：探测目标原路径是否已被重新占用；若无冲突，按逆序逐一移回，保证系统状态 100% 恢复。
