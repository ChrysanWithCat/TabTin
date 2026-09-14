# 项目架构 FAQ：Django + Celery + PostgreSQL + Redis + Centrifugo + OSS

> 基于当前仓库静态分析整理，重点回答：这些组件分别是什么、在 TabTin 项目里承担什么角色、哪些功能链路用到了它们。  
> 主要参考文件：`compose.yaml`、`docker-compose.dev.yml`、`apps/tabtin_django/tabtin/settings.py`、`apps/tabtin_django/tabtin/celery.py`、`apps/tabtin_django/tabtin/runtime/registry.py`、`apps/tabtin_django/apps/services/oss/`、`apps/tabtin_django/apps/tabchat/`、`apps/collab-live/`、`apps/tabtin-electron/`。

---

## 0. 一句话总览

| 组件 | 一句话解释 | 在 TabTin 中的核心定位 |
| --- | --- | --- |
| **Django** | Python Web 后端框架 | 业务 API、权限、组织/空间/会话/文件等核心业务控制面 |
| **Celery** | Python 异步任务队列 | 后台任务、定时任务、耗时任务、重试任务执行层 |
| **PostgreSQL** | 关系型数据库 | 权威业务数据库，保存用户、组织、会话、文件元数据、任务结果、索引/向量等持久数据 |
| **Redis** | 内存型键值存储 | 队列 Broker、缓存、分布式锁、限流、实时通道辅助、协同服务扩展支撑 |
| **Centrifugo** | 实时消息 / WebSocket PubSub 服务 | 面向客户端的实时推送层，用于 IM、AI 流式消息、空间事件、在线状态等 |
| **OSS** | Object Storage Service，对象存储 | 保存大文件、图片、附件、导入导出产物、诊断包、更新包等二进制对象 |

可以把它们理解成：

```text
客户端 Electron / Web / Mobile
        │
        ▼
Django API / 业务控制面  ───────► PostgreSQL：权威业务数据
        │                         Redis：缓存、锁、队列 Broker、实时基础设施
        │
        ├────► Celery Workers：异步/定时/耗时任务
        │            │
        │            ├────► PostgreSQL：写任务状态、业务结果
        │            ├────► OSS：读写文件和产物
        │            └────► Centrifugo：必要时推送实时结果
        │
        ├────► OSS：上传、下载、预签名、文件登记
        │
        └────► Centrifugo：向客户端推送聊天、AI、空间事件
                         ▲
                         │ WebSocket
客户端订阅 chat:* / personal:* / space:* 等频道
```

---

## Q1：Django 是什么？在项目中起什么作用？哪些功能用到了？

### Django 是什么？

Django 是一个 Python Web 后端框架，通常负责：

- 暴露 HTTP API；
- 执行业务逻辑；
- 管理用户、权限、组织、会话等业务对象；
- 通过 ORM 读写数据库；
- 接入中间件、认证、路由、后台管理、任务调度配置等。

本项目后端位于：

- `apps/tabtin_django/`

配置入口主要在：

- `apps/tabtin_django/tabtin/settings.py`
- `apps/tabtin_django/tabtin/urls.py`
- `apps/tabtin_django/tabtin/urls_deferred.py`

### 在 TabTin 中的作用

Django 是 TabTin 的**业务后端和控制面**。它不是只做一个简单 API 网关，而是承载大量核心业务能力：

1. **用户、认证、组织与权限**
   - 登录、JWT、用户状态、组织/成员、工作空间权限等。
2. **TabTin 核心业务 API**
   - `tabdoc`、`tabdata`、`tabslide`、`tabmemo`、`tabsite`、`tins`、`tabchat`、`tabtinspace` 等模块。
3. **Agent / Skill / Capability / Memory 等 AI 协作能力**
   - Agent 编排、技能、能力注册、凭据保险箱、用户画像、Agent Memory 等。
4. **OSS 文件服务 API**
   - 上传配置、预签名上传、确认上传、下载、文件列表、存储统计等。
5. **实时系统的认证与业务校验**
   - Centrifugo 连接、订阅、刷新代理接口；判断用户是否能订阅 `chat:*`、`space:*`、`personal:*` 等频道。
6. **异步任务的定义与调度入口**
   - Celery app 初始化、任务自动发现、Beat 定时任务注册。
7. **运维与管理 API**
   - health、admin、metrics、updater、diagnostics、client_errors 等。

### 哪些功能用到了 Django？

几乎所有需要业务规则的功能都会经过 Django，例如：

- 用户登录、鉴权、组织成员与空间权限；
- 聊天、会话、AI mention、消息投递 outbox；
- Agent、Skills、Capabilities、Credential Vault；
- TabDoc、TabData、TabSlide、TabMemo、TabSite、Tins；
- 文件上传下载、OSS 元数据登记、存储用量统计；
- 账单、会员、钱包、支付、额度；
- RAG、FTS、搜索、向量检索相关元数据；
- Feishu、GitHub 等外部集成；
- Centrifugo 连接/订阅鉴权；
- Collab Live 协同编辑相关权限验证、快照/持久化接口。

---

## Q2：Celery 是什么？在项目中起什么作用？哪些功能用到了？

### Celery 是什么？

Celery 是 Python 生态里常用的异步任务队列。它解决的问题是：

- 有些事情不适合在 HTTP 请求里同步完成；
- 有些任务很耗时，比如导入、解析、生成、索引、清理；
- 有些任务需要失败重试；
- 有些任务需要定时执行。

Celery 通常由三部分组成：

```text
Django 投递任务  →  Broker 队列，例如 Redis  →  Celery Worker 执行任务
```

本项目中相关依赖包括：

- `celery==5.3.4`
- `django-celery-beat==2.5.0`
- `django-celery-results==2.5.1`

入口：

- `apps/tabtin_django/tabtin/celery.py`
- `apps/tabtin_django/tabtin/runtime/registry.py`

### 在 TabTin 中的作用

Celery 是 TabTin 的**后台执行层**。它负责把不适合在 API 请求中同步执行的工作移到后台，例如：

1. **耗时任务**
   - 文档解析、PPTX 生成、文件导入导出、LLM 异步调用、媒体处理等。
2. **定时任务**
   - 清理、索引、账单/会员/钱包状态刷新、任务巡检、运行时维护等。
3. **可靠投递与重试**
   - IM outbox、channel gateway 投递、失败重试、去抖/补偿任务。
4. **后台 AI 工作**
   - Agent Memory 提取、压缩、会话总结、日记生成、空闲结算等。

仓库里的 `runtime/registry.py` 对队列有清晰分层，例如：

| 队列类型 | 主要用途 |
| --- | --- |
| `critical` | 支付、会员、钱包、短信等高优先级任务 |
| `default` | 普通后台任务 |
| `heavy` | 大文件、导入导出、PPTX、LLM 等重任务 |
| `docparse` | 文档解析、导入解析 |
| `realtime_delivery` | 实时消息 outbox 投递与重试 |
| `search_indexing` | 全文检索、索引构建 |
| `tracker_agent` | Tracker / Agent 相关后台任务 |
| `ai_background` | Agent memory、会话总结、日记等 AI 后台任务 |
| `media` | 媒体生成、结果轮询、结果落库 |

### 哪些功能用到了 Celery？

典型使用场景：

- **文件与 OSS**
  - 异步上传、URL 下载后上传、批量下载上传、临时文件清理、存储统计维护。
- **TabDoc / 文档解析**
  - 文档导入、解析、HTML/图片/附件处理、缺失二进制修复、清理。
- **TabSlide**
  - PPTX 生成、导入、历史处理、字体迁移、媒体资源处理。
- **TabData**
  - 数据导入导出、开放存储、附件处理。
- **TabChat / IM**
  - 消息 outbox 投递、AI 流式/最终/错误事件推送、会话后台处理。
- **RAG / FTS / Search**
  - 索引构建、检索元数据更新、向量/全文索引任务。
- **Agent / AI 背景任务**
  - 记忆抽取、压缩、任务总结、用户画像、日记。
- **会员、钱包、支付、短信**
  - 高优先级、可重试、需要可靠执行的业务任务。
- **Channel Gateway / 集成**
  - 入站/出站消息去抖、补偿、重试投递。

---

## Q3：PostgreSQL 是什么？在项目中起什么作用？哪些功能用到了？

### PostgreSQL 是什么？

PostgreSQL 是一个成熟的关系型数据库，适合保存权威业务数据。它支持事务、索引、JSON、全文搜索、行锁等能力，也可以通过扩展支持向量检索。

本项目使用：

- Docker 镜像：`pgvector/pgvector:pg16`
- Python 驱动：`psycopg2-binary==2.9.9`
- 向量扩展：`pgvector==0.2.4`

在 `compose.yaml` 中，PostgreSQL 服务名为：

- `postgres`

默认端口：

- `5432`

### 在 TabTin 中的作用

PostgreSQL 是 TabTin 的**权威业务数据库**。也就是说：

- Redis 可以丢缓存，Celery 可以重试任务，Centrifugo 可以重新连接；
- 但最终业务事实，例如用户、消息、文件记录、空间、订单、会员状态、索引元数据等，主要应该落在 PostgreSQL。

`apps/tabtin_django/tabtin/settings.py` 中可以看到项目默认使用 `single_pg` 模式，即单 PostgreSQL 承载主要业务数据：

```text
TABTIN_DATABASE_MODE = single_pg
```

项目中还配置了很多数据库路由/模块映射，覆盖：

- `tabtinspace`
- `tabdoc`
- `tracker`
- `tabdata`
- `rag`
- `tabslide`
- `tabcode`
- `tabmemo`
- `agent_memory`
- `user_portrait`
- `extensions`
- `tabchat`
- `collab`
- `capabilities`
- `client_errors`
- `tins`
- `skills`
- `tabsite`
- 等等。

### 哪些功能用到了 PostgreSQL？

基本所有需要持久化的功能都会用到 PostgreSQL：

- **账号与组织**：用户、组织、成员、空间、权限；
- **聊天与协作**：Conversation、Message、会话状态、IMEventOutbox；
- **TabDoc / TabData / TabSlide / TabMemo / TabSite / Tins**：业务对象、版本、历史、元数据；
- **Agent / Skills / Capabilities**：Agent 配置、技能、能力、工具 embedding、权限；
- **RAG / Search / FTS**：索引任务、检索元数据、全文检索、向量字段；
- **OSS 文件系统**：`FileRecord`、`UploadTask`、`FileUsage`、存储分析；
- **账单/支付/会员/钱包**：订单、额度、交易、订阅状态；
- **Celery 结果与定时任务**：`django-celery-results` 保存任务结果，`django-celery-beat` 保存定时任务配置；
- **运维诊断**：client errors、diagnostics、updater、平台配置等。

### PostgreSQL 在项目里的几个关键特性

- **事务与行锁**：用于 outbox、任务领取、并发状态更新等；
- **JSONField**：保存灵活业务配置、状态、上下文；
- **全文搜索**：例如 `SearchVector` 相关能力；
- **pgvector**：能力/工具 embedding、语义检索等；
- **统一数据源**：在社区部署中通过 `single_pg` 降低多数据库复杂度。

---

## Q4：Redis 是什么？在项目中起什么作用？哪些功能用到了？

### Redis 是什么？

Redis 是一个高性能内存型 Key-Value 存储。它常用于：

- 缓存；
- 分布式锁；
- 计数器与限流；
- 消息队列 Broker；
- Pub/Sub；
- 短期状态与临时数据。

本项目使用：

- Docker 镜像：`redis:8-alpine`
- Python 客户端：`redis==5.0.1`
- Django Cache：`django-redis==5.4.0`
- Channels Redis：`channels-redis==4.3.0`

### 在 TabTin 中的作用

Redis 是 TabTin 的**高速临时基础设施层**，它不应该被理解为权威业务数据库，而是承载“快、短期、可重建”的数据与通道。

`compose.yaml` / `settings.py` 中有明显的 Redis 分工：

| Redis 用途 | 默认 DB / URL | 作用 |
| --- | --- | --- |
| Celery Broker | `redis://redis:6379/0` | Celery 任务队列投递 |
| Django Cache | `redis://redis:6379/1` | 缓存、计数、限流、临时状态 |
| Channels Layer | `redis://redis:6379/2` | Django Channels 通道层 |
| Centrifugo | 依赖 Redis 服务 | 实时服务扩展、Pub/Sub、presence 等支撑 |
| Collab Live | `ioredis` / Hocuspocus Redis 扩展 | 多实例 Yjs 协同同步 |

### 哪些功能用到了 Redis？

典型功能包括：

- **Celery 任务队列**
  - Django 把任务投给 Redis，Celery Worker 从 Redis 拉取任务执行。
- **Django 缓存**
  - 高频配置、临时状态、业务缓存、运行时状态。
- **限流与验证码**
  - 登录/认证验证码、邀请、凭据查看、敏感操作节流等。
- **分布式锁与并发保护**
  - 防止同一资源被重复处理，例如投递去重、任务并发保护、存储配额预占等。
- **实时通道辅助**
  - Django Channels 的 Redis channel layer；Centrifugo 的扩展/状态支撑。
- **协同编辑服务**
  - `apps/collab-live` 里的 Hocuspocus/Yjs 多实例同步。
- **聊天与 AI 流式输出保护**
  - 消息序列、stream guard、去抖、重连后的状态修复等。
- **后台运行态**
  - billing guard、degradation state、agent 执行缓存、设备在线状态等。

### Redis 和 PostgreSQL 的区别

| 对比项 | PostgreSQL | Redis |
| --- | --- | --- |
| 数据性质 | 权威、持久、可审计 | 临时、高速、可重建 |
| 常见内容 | 用户、消息、订单、文件记录、索引元数据 | 缓存、锁、限流、队列、短期状态 |
| 是否适合长期保存业务事实 | 是 | 不适合 |
| 是否强调速度和低延迟 | 较高，但不是内存级 | 非常高 |

---

## Q5：Centrifugo 是什么？在项目中起什么作用？哪些功能用到了？

### Centrifugo 是什么？

Centrifugo 是一个面向实时场景的 WebSocket / PubSub 服务。它的典型职责是：

- 客户端建立 WebSocket 长连接；
- 客户端订阅频道；
- 后端向频道发布事件；
- Centrifugo 把事件实时推给订阅客户端；
- 支持 presence、断线重连、刷新 token、频道权限等机制。

本项目使用：

- Docker 镜像：`centrifugo/centrifugo:v6`
- 客户端依赖：`apps/tabtin-electron/package.json` 中的 `centrifuge`
- 默认端口：`8100`

### 在 TabTin 中的作用

Centrifugo 是 TabTin 的**客户端实时消息层**。它解决的问题是：

- 聊天消息需要实时到达；
- AI 回复需要边生成边展示；
- 空间/团队事件需要及时同步；
- 在线状态、presence、连接状态需要低延迟更新；
- 这些事情不适合只靠普通 HTTP 轮询。

Django 负责业务判断和发事件，Centrifugo 负责高并发 WebSocket 连接与频道广播：

```text
Django / Celery 产生事件
        │ HTTP API publish/broadcast
        ▼
Centrifugo
        │ WebSocket
        ▼
Electron / Web 客户端订阅频道并接收实时消息
```

相关 Django 代码：

- `apps/tabtin_django/apps/tabchat/services/centrifugo_service.py`
- `apps/tabtin_django/apps/tabchat/centrifugo_proxy.py`

相关前端代码：

- `apps/tabtin-electron/src/renderer/src/hooks/useCentrifugoClient.ts`

### 哪些功能用到了 Centrifugo？

主要包括：

- **TabChat / IM 实时聊天**
  - 发送消息后推送到 `chat:{conversation_id}`；
  - 个人通知推送到 `personal:{user_id}`。
- **AI mention / Agent 回复流式展示**
  - AI token streaming、最终消息、错误事件投递到聊天频道。
- **IM Outbox 可靠投递**
  - 先把事件写入 PostgreSQL outbox，再由 Celery `realtime_delivery` worker 投递到 Centrifugo。
- **空间事件与团队协作事件**
  - `space:{space_id}` 相关频道事件。
- **在线状态与 presence**
  - 客户端连接状态、团队空间在线成员、presence stats。
- **组织/成员变更后的连接控制**
  - 成员退出、权限变更时，可以 disconnect 或 unsubscribe。
- **Electron 实时 UI**
  - 聊天列表、消息流、连接状态、重连后缓存刷新等。

### Centrifugo 和 Django Channels / Redis Channel Layer 的区别

项目中同时配置了 Django Channels + Redis channel layer，但它们和 Centrifugo 的定位不同：

| 组件 | 面向谁 | 主要作用 |
| --- | --- | --- |
| **Centrifugo** | 面向客户端 | 大量 WebSocket 连接、频道订阅、实时推送 |
| **Django Channels + Redis** | 面向 Django 内部/ASGI 能力 | Django 内部通道层、异步通信基础设施 |
| **Redis** | 被多个系统依赖 | 为 Celery、缓存、Channels、Centrifugo、Collab Live 提供底层支撑 |

---

## Q6：OSS 是什么？在项目中起什么作用？哪些功能用到了？

### OSS 是什么？

OSS 是 Object Storage Service，即对象存储。它适合保存：

- 图片；
- 附件；
- 文档原始文件；
- 导入导出产物；
- 压缩包；
- 诊断包；
- 更新包；
- AI/媒体生成结果等大体积二进制对象。

注意：这里的 OSS 是一个**对象存储抽象层**，不等同于只支持阿里云 OSS。

在当前项目中，OSS 有两种典型提供方：

| 模式 | Provider | 用途 |
| --- | --- | --- |
| 社区版 / 本地开发 | `local` | 文件保存在本地目录或 Docker volume 中，例如 `/var/lib/tabtin/objects` |
| 生产 / 云部署 | `aliyun` | 使用阿里云 OSS，配置 `ALIYUN_OSS_*` |

相关配置在：

- `apps/tabtin_django/tabtin/settings.py`

相关服务代码在：

- `apps/tabtin_django/apps/services/oss/`

### 在 TabTin 中的作用

OSS 是 TabTin 的**文件和二进制对象存储层**。项目不会把大文件本体直接塞进 PostgreSQL，而是采用：

```text
文件本体      → OSS / Local Object Storage
文件元数据    → PostgreSQL FileRecord / FileUsage / UploadTask
上传下载控制  → Django OSS API
耗时搬运处理  → Celery OSS tasks
```

这可以带来几个好处：

- 数据库不会被大文件撑爆；
- 上传下载可以走预签名 URL，减轻后端压力；
- 文件可以统一登记、统计、计费、授权；
- 不同部署环境可以切换本地存储或云对象存储；
- 文档、PPT、图片、附件、导出包等可以复用同一存储抽象。

### 哪些功能用到了 OSS？

#### 1. 通用文件服务

代码中有完整 OSS API，例如：

- 上传配置：`/upload-config`
- 本地上传：`/local-upload`
- 上传：`/upload`
- 下载：`/download/{file_id}`
- 预签名 URL：`/presigned-url`
- 批量上传：`/batch-upload`
- 预签名上传：`/presign-upload`
- 确认上传：`/confirm-upload`
- 存储统计：`/storage/overview`、`/storage/by-module`、`/storage/by-member` 等。

#### 2. 文档/知识/办公类功能

- **TabDoc**：图片、附件、HTML artifact、docx 转换产物、文档二进制；
- **TabSlide**：PPTX 导入/导出、缩略图、字体、媒体素材、缓存产物；
- **TabData**：附件、导入文件、导出文件、开放存储；
- **TabMemo / TabSite / Tins**：内容附件、静态资源、导出产物；
- **RAG / 文档解析**：临时解析文件、索引前原始材料。

#### 3. 聊天与 AI 功能

- 聊天附件、共享资源；
- 会话续接资源；
- Agent / Skill 包资源；
- 媒体生成结果落库；
- AI 处理过程中产生的中间文件或最终产物。

#### 4. 平台与运维功能

- Updater 更新包、manifest；
- Diagnostics 诊断包；
- Feishu 导入的图片/文件；
- package registry / skills 资源包；
- 存储用量统计、配额、清理、按模块/成员分析。

---

## Q7：一个典型功能链路中，这些组件如何串起来？

### 链路 1：用户上传文件并触发文档解析

```text
1. 客户端请求 Django：获取上传配置或预签名 URL
2. 文件上传到 OSS：本地对象存储或阿里云 OSS
3. Django 在 PostgreSQL 写 FileRecord / UploadTask / FileUsage
4. Django 投递 Celery 任务：文档解析、转换、索引
5. Celery 从 Redis Broker 拉任务执行
6. Celery 读 OSS 文件，解析后写 PostgreSQL，并可能产生新文件再写 OSS
7. 处理进度/结果通过 Centrifugo 推给客户端，或客户端通过 Django API 查询
```

组件分工：

| 步骤 | 组件 |
| --- | --- |
| API、权限、业务编排 | Django |
| 原始文件/产物保存 | OSS |
| 文件记录/任务状态/业务结果 | PostgreSQL |
| 异步任务排队 | Redis |
| 异步任务执行 | Celery |
| 实时进度通知 | Centrifugo |

### 链路 2：用户发送聊天消息，AI 实时回复

```text
1. 客户端调用 Django TabChat API 发送消息
2. Django 校验权限，并把 conversation/message 写入 PostgreSQL
3. Django 触发 AI mention / Agent 处理，必要时投递 Celery 后台任务
4. AI 生成过程中的 stream/final/error 事件写入或投递
5. IM 可靠事件可先写 PostgreSQL Outbox
6. Celery realtime_delivery worker 从 Outbox 取事件
7. Django/Celery 调用 Centrifugo API 发布到 chat:{conversation_id}
8. Electron/Web 客户端通过 WebSocket 收到实时消息并更新 UI
```

组件分工：

| 步骤 | 组件 |
| --- | --- |
| 聊天 API、权限、业务状态 | Django |
| Conversation / Message / Outbox | PostgreSQL |
| AI/投递后台任务 | Celery |
| 任务队列与短期状态 | Redis |
| 实时推送 | Centrifugo |
| 附件/生成资源 | OSS |

### 链路 3：多人协同编辑 TabDoc / TabData / TabSlide

```text
1. 客户端进入协同编辑页面
2. Collab Live 提供 Yjs WebSocket 协同能力
3. Django 校验用户对空间/文档的访问权限
4. 协同状态、快照、业务元数据落 PostgreSQL
5. Redis 支撑 Collab Live 多实例同步
6. 图片、附件、PPTX、导入导出产物落 OSS
7. 重要空间/成员/状态事件可通过 Centrifugo 推送
```

补充说明：`apps/collab-live` 是 Hocuspocus/Yjs WebSocket 服务，不在用户列出的六个组件中，但它和 Redis、Django、PostgreSQL、OSS 一起支撑实时协同编辑。

### 链路 4：RAG / 搜索 / 能力匹配

```text
1. Django 接收业务数据或上传材料
2. 原始材料进入 OSS，元数据进入 PostgreSQL
3. Django 投递 Celery 索引任务
4. Celery 构建全文索引、向量、RAG 元数据
5. PostgreSQL / pgvector 保存检索相关数据
6. 查询时 Django API 读取索引结果并返回给客户端或 Agent
```

---

## Q8：这六个组件之间怎么区分？

| 问题 | 应该看哪个组件 |
| --- | --- |
| 这个 API 是谁提供的？ | Django |
| 业务权限是谁判断的？ | Django |
| 用户、消息、订单、文件记录存在谁那里？ | PostgreSQL |
| 大文件、图片、附件存在谁那里？ | OSS |
| 耗时任务谁执行？ | Celery |
| Celery 任务排队靠谁？ | Redis |
| 缓存、锁、限流、临时状态靠谁？ | Redis |
| 客户端实时收到聊天/AI/空间事件靠谁？ | Centrifugo |
| 多人协同编辑的底层实时文档同步靠谁？ | Collab Live / Yjs，Redis 辅助扩展，Django 做权限与持久化 |

---

## Q9：本地 / 社区部署中它们分别跑在哪里？

从 `compose.yaml` 看，社区部署会启动这些服务：

| 服务名 | 镜像/来源 | 默认端口 | 说明 |
| --- | --- | --- | --- |
| `django` | `tabtin/community-django:local` | `6060` | 后端 API 服务 |
| `celery` | 同 Django 镜像，执行 worker 命令 | 不直接暴露 | 后台 Worker |
| `postgres` | `pgvector/pgvector:pg16` | `5432` 容器内 | PostgreSQL + pgvector |
| `redis` | `redis:8-alpine` | `6379` 容器内 | Broker/cache/channel 支撑 |
| `centrifugo` | `centrifugo/centrifugo:v6` | `8100` | 实时 WebSocket 服务 |
| `local-objects` volume | 本地 Docker volume | 不直接暴露 | local OSS 文件对象存储 |

`compose.lan.yaml` 会把 Django 和 Centrifugo 暴露到局域网：

- Django：`0.0.0.0:6060:6060`
- Centrifugo：`0.0.0.0:8100:8100`

开发环境 `docker-compose.dev.yml` 更偏向“只起基础设施”，例如 PostgreSQL 和 Redis，Django / 前端服务可在宿主机运行。

---

## Q10：如果系统出问题，如何按组件定位？

| 现象 | 优先排查 |
| --- | --- |
| API 访问失败、登录失败、权限判断异常 | Django 日志、Django 配置、数据库连接 |
| 页面能打开但消息不实时刷新 | Centrifugo 连接、token/订阅代理、客户端订阅、Redis/Centrifugo 状态 |
| 后台处理一直不完成 | Celery worker、Redis Broker、任务队列、任务日志 |
| 上传失败、下载失败、图片/附件丢失 | OSS provider、local object volume、Aliyun OSS 配置、FileRecord |
| 数据丢失或业务状态异常 | PostgreSQL 数据、迁移、事务、模型逻辑 |
| 性能抖动、高频接口慢 | Redis cache、数据库索引、Celery 队列堆积、Django 慢查询 |
| 协同编辑不同步 | Collab Live、Redis 扩展、Django 权限/快照接口、客户端 WebSocket |

---

## Q11：面向分享时可以怎么讲？

可以用下面这个类比：

> Django 像“公司前台 + 业务中台”，负责接请求、查权限、定规则；  
> PostgreSQL 像“档案室”，保存最终可信的业务记录；  
> Redis 像“白板和传送带”，放缓存、锁、验证码、任务队列这些短期信息；  
> Celery 像“后台工人”，专门处理耗时、定时、可重试的活；  
> Centrifugo 像“实时广播系统”，把聊天、AI 回复、空间事件推给在线客户端；  
> OSS 像“文件仓库”，保存图片、附件、PPT、导入导出包和其他大文件。

如果只用一句话概括 TabTin 的后端架构：

> Django 管业务，PostgreSQL 存事实，Redis 做高速中间层，Celery 跑后台任务，Centrifugo 推实时消息，OSS 存大文件。

---

## 附：仓库证据索引

| 证据点 | 文件/目录 |
| --- | --- |
| 服务编排：PostgreSQL、Redis、Django、Celery、Centrifugo、local objects | `compose.yaml` |
| 开发环境基础设施 | `docker-compose.dev.yml` |
| Django、数据库、Redis、Celery、Centrifugo、OSS 配置 | `apps/tabtin_django/tabtin/settings.py` |
| Celery 初始化与 Beat schedule 汇总 | `apps/tabtin_django/tabtin/celery.py` |
| 队列/Worker/Beat/Runtime Registry | `apps/tabtin_django/tabtin/runtime/registry.py` |
| OSS 模型、API、任务、provider 抽象 | `apps/tabtin_django/apps/services/oss/` |
| Centrifugo 服务封装与代理鉴权 | `apps/tabtin_django/apps/tabchat/services/centrifugo_service.py`、`apps/tabtin_django/apps/tabchat/centrifugo_proxy.py` |
| Electron 端 Centrifugo 客户端 | `apps/tabtin-electron/src/renderer/src/hooks/useCentrifugoClient.ts` |
| Collab Live / Yjs 协同服务 | `apps/collab-live/` |
| Python 依赖版本 | `apps/tabtin_django/requirements.txt` |
| Electron Centrifuge 客户端依赖 | `apps/tabtin-electron/package.json` |

---

## Q12：既然 Centrifugo 已经是实时消息，WebSocket Gateway（AsyncAPI 契约）这个实时通道是干嘛的？

先拆开两个名词：

- **WebSocket Gateway**：项目自己实现的 Django ASGI WebSocket 入口，路由是 `ws/v1/gateway`，代码入口是 `apps/tabtin_django/tabtin/routing.py` 和 `apps/tabtin_django/apps/services/common/ws/gateway.py`。
- **AsyncAPI 契约**：不是一个运行中的服务，而是这个 WebSocket Gateway 的协议说明书，文件是 `contracts/asyncapi/ws-gateway.yaml`。它规定连接地址、角色、能力、消息 envelope、topic、错误码、心跳、订阅、恢复等格式。

一句话区别：

> **Centrifugo 更像“IM/频道广播基础设施”；WebSocket Gateway 更像“TabTin 自己的 Agent / 设备 / 协作控制总线”。**

### 1. WebSocket Gateway 主要解决什么问题？

它不是单纯为了“把消息推给客户端”，而是为了让 TabTin 多端、本地 Daemon、云端 Django、Agent Runtime 之间有一条**统一、可鉴权、可订阅、可恢复、可双向通信的协议通道**。

它承担的事情包括：

1. **Agent 运行时事件流**
   - `agent.stream.*`：Agent 输出流、工具调用、步骤、done、message_persisted、subagent 进度等。
   - Electron / Web / Mobile 可以订阅对应 topic 看任务进展。

2. **设备/Daemon 控制面**
   - `agent.action.device.{fingerprint}`：云端把需要在某台设备执行的 action 下发给本地 Daemon/Electron。
   - 本地执行完后通过 `agent.action.result` 回传结果。
   - 这就是“手机发起任务，桌面电脑真正执行”的关键通道之一。

3. **HITL / 人在回路审批**
   - `agent.action.approval_request`
   - `agent.action.approval_response`
   - `localrt.user_response`
   - 用来处理需要用户确认、授权、补充输入的流程。

4. **上下文与协作事件同步**
   - `context.sync.*`
   - `table.events.*`
   - `doc.events.*`
   - `session.collaboration.*`
   - 例如新建/修改/归档文档后，侧边栏、空间列表、资源视图自动刷新。

5. **客户端/设备状态上报**
   - `device.status`
   - `device.capabilities.report`
   - `device.capabilities.refresh.*`
   - `git.status.report`
   - `git.diff.request / response`
   - 让云端知道某个设备是否在线、具备哪些能力、Git 状态是什么。

6. **语音、通知、账单、扩展等事件流**
   - `asr.stream.*`
   - `tts.stream.*`
   - `notifications`
   - `billing.events`
   - `extension.events`
   - `tracker.events`

7. **弱网/断线恢复**
   - `resume` 协议；
   - 事件写入 Redis Stream buffer；
   - 客户端重连后可以按 `last_event_id` / topic cursor 补拉错过的事件。

### 2. 它和 Centrifugo 的区别

| 对比项 | Centrifugo | WebSocket Gateway |
| --- | --- | --- |
| 本质 | 独立实时 Pub/Sub 服务 | Django ASGI + Channels 实现的自研 WebSocket 协议入口 |
| 运行端口 | 通常是 `8100`，路径类似 `/connection/websocket` | Django 后端 `6060` 下的 `/ws/v1/gateway` |
| 协议形态 | Centrifugo 自带协议，客户端订阅 channel | TabTin 自定义 `GatewayEnvelope`，由 AsyncAPI 契约描述 |
| 主要用途 | IM、聊天频道、个人通知、空间实时广播 | Agent 流、设备控制、Daemon 通信、上下文同步、表格/文档事件、Git/能力/审批等 |
| 通信方向 | 更偏“服务端发布 → 客户端订阅” | 强双向：客户端/Daemon 可发命令、上报结果，服务端也可下发 action/事件 |
| 鉴权模型 | Django 通过 Centrifugo proxy 判断连接/订阅权限 | Gateway 自己做 auth、role、capability、topic 权限校验 |
| 可靠性补偿 | TabChat 侧有 IM outbox + Celery 投递 | Gateway 侧有 Redis Stream event buffer + `resume` 重放 |
| 典型 topic | `chat:{conversation_id}`、`personal:{user_id}`、`space:{space_id}` | `agent.stream.{id}`、`agent.action.device.{fingerprint}`、`context.sync.*`、`table.events.*` 等 |

### 3. 为什么不只用 Centrifugo？

因为 WebSocket Gateway 要承载的不是普通聊天广播，而是更“业务协议化”的东西：

- 需要区分 `electron`、`mobile`、`daemon`、`admin`、`channel`、`backend` 等角色；
- 需要声明 capability，比如 `agent.stream`、`agent.action`、`context.sync`、`table.events`；
- 需要允许客户端/Daemon 向服务端发业务消息，例如 action result、Git 状态、relay_events、chat.cancel；
- 需要和 Django 的业务权限、设备绑定、组织边界、会话状态深度结合；
- 需要跨端契约，TS / Kotlin / Swift / Python 都能按同一套事件类型对齐；
- 需要断线恢复、事件 replay、按 topic cursor 补偿。

这些事情如果全部塞到 Centrifugo 的普通频道 Pub/Sub 里，会让 Centrifugo 频道变成“半个业务网关”，权限、协议、回放、设备路由都会变复杂；所以项目把它们拆成两套实时系统。

### 4. 一个典型 Agent 链路

```text
手机 / Web / Electron 发起 Agent 任务
        │
        ▼
Django 创建会话、检查权限、决定目标设备
        │
        ▼
WebSocket Gateway 下发到 agent.action.device.{fingerprint}
        │
        ▼
本地 Daemon / Electron AgentHost 执行文件、终端、浏览器等动作
        │
        ▼
Daemon / Electron 通过 WebSocket Gateway 回传 agent.action.result / relay_events
        │
        ▼
Django 持久化关键状态，并通过 agent.stream.* topic 推给正在观看的客户端
```

这里的重点是：

- **执行发生在本地设备**；
- **云端 Django 做调度、权限、持久化、协作同步**；
- **WebSocket Gateway 连接云端和本地运行时**；
- **客户端看到的 Agent 流和状态也经这个协议统一分发**。

### 5. 一个典型 IM 链路

```text
用户发聊天消息
        │
        ▼
Django TabChat API 写入 PostgreSQL
        │
        ▼
IM 事件进入 outbox / Celery 投递
        │
        ▼
Django 调 Centrifugo API publish 到 chat:{conversation_id}
        │
        ▼
客户端通过 Centrifugo WebSocket 收到消息
```

这里更像传统 IM：频道订阅、消息广播、在线用户实时收到。

### 6. 所以应该怎么记？

可以这样讲：

> TabTin 有两类实时通道：  
> **Centrifugo 负责“聊天/通知型实时广播”**；  
> **WebSocket Gateway 负责“Agent/设备/协作协议型实时总线”**。

再压缩成一句：

> **Centrifugo 是实时消息中间件；WebSocket Gateway 是 TabTin 自己定义的跨端实时业务协议。**

补充：仓库里的 `apps/channel_gateway/` 名字也叫 Gateway，但它主要是 Slack、Feishu、Discord、钉钉、Teams、Mattermost 等**外部沟通渠道接入层**，不要和这里的 `/ws/v1/gateway` 混淆。

## Q13：这个项目的沙箱是内核级沙箱，还是只是应用层 judge？

结论：**两层都有，但要分清边界**。

更准确地说：TabTin 不是只靠一个应用层 judge，也不是所有场景都运行在 Firecracker / gVisor 这类强隔离虚拟化沙箱里。仓库里能看到的是：

1. **应用层 policy judge 是第一道也是最核心的权限闸门**：决定某个工具调用 / 终端命令是 `allow`、`ask` 还是 `deny`；
2. **终端命令在特定 `route=sandbox` 策略下，会尝试套一层 OS 级沙箱**：Linux 用 `bubblewrap(bwrap)`，macOS 用 `sandbox-exec(Seatbelt)`，WSL2 走 Linux bwrap；
3. **Windows 原生环境目前会降级**：没有真正 OS 级沙箱，代码明确写着安全性主要靠 denylist / allowlist / 路径边界等应用层规则保障。

所以如果别人问“是不是内核级沙箱”，不要简单回答“是”或“不是”。推荐说：

> **TabTin 的沙箱模型是“应用层策略判定 + 可选 OS 级进程沙箱”的组合。Linux/macOS/WSL2 的终端 sandbox 路径会用到操作系统隔离能力；但 Windows 原生会降级，而且它不是 Firecracker/gVisor 那种强虚拟化沙箱。**

### 1. 应用层 judge 在做什么？

应用层 judge 主要在 `packages/security-policy/`，核心文件是：

- `packages/security-policy/src/judge.ts`
- `packages/security-policy/src/types-v3.ts`
- `packages/security-policy/src/build-policy.ts`
- `packages/agent-runtime/src/engine/tooling/tool-orchestration.ts`

它的角色是：**工具真正执行之前，先判断这一步是否应该被允许**。

它关心的不是“把进程关进哪个内核 namespace”，而是业务安全策略，例如：

- 当前 Agent 的审批档位：`always_ask`、`auto`、`full_access`；
- 当前模式是否允许工具调用：Ask / Plan / Study / Agent 等模式下能不能写文件、改计划、操作设备；
- 工具类型：`file`、`shell`、`object_read`、`object_write`、`mcp`、`device`；
- 路径是否在 workspace / sandbox / sessionApprovedPaths 里；
- 是否命中 hardline 红线命令或红线路径；
- 是否访问 `.ssh`、凭据、系统敏感路径等；
- 是否需要用户审批并生成 approval key / memo；
- 是否走 `allow`、`ask`、`deny`。

典型链路是：

```text
LLM 产生 tool call
        │
        ▼
agent-runtime/tool-orchestration
        │
        ▼
security-policy/judge.ts 做 allow / ask / deny 判定
        │
        ├─ deny：直接返回错误，不执行
        ├─ ask：走 HITL 用户审批
        └─ allow：才进入真正工具或终端执行
```

这里的 “judge” 本质是**应用层授权裁判**，属于 policy sandbox / permission gateway。

### 2. OS 级沙箱在哪里？

OS 级沙箱主要在 `packages/terminal-core/`，核心文件是：

- `packages/terminal-core/src/commandExecutor.ts`
- `packages/terminal-core/src/platform/index.ts`
- `packages/terminal-core/src/platform/linux.ts`
- `packages/terminal-core/src/platform/darwin.ts`
- `packages/terminal-core/src/platform/windows.ts`
- `packages/terminal-core/src/backend/spawn-sandbox-backend.ts`

当终端执行策略要求 `route=sandbox`，并且带有 `sandboxLevel` 时，`CommandExecutor` 会：

1. 为本次 thread 准备一个 sandbox 目录；
2. 把项目目录复制到 sandbox project 目录，过滤 `.git`、`node_modules`、`.venv`、`dist` 等；
3. 把复制后的 project 目录设为只读；
4. 准备一个可写 tmp 目录；
5. 如果当前平台有可用 OS 沙箱，就用 OS 沙箱包装 `spawn()` 执行命令；
6. 如果 OS 沙箱不可用，则标记 `osSandboxDegraded`，降级执行。

两个 sandbox level 的语义在 `packages/terminal-core/src/types.ts` 中定义：

- `filesystem`：文件系统隔离，网络默认放行；
- `complete`：文件系统隔离 + 网络隔离。

### 3. Linux 上是不是内核级？

Linux 这条路径相对接近“内核级沙箱”，因为它使用 `bubblewrap(bwrap)`，而 bwrap 背后依赖 Linux namespace / mount namespace / user namespace 等内核能力。

在 `packages/terminal-core/src/platform/linux.ts` 里可以看到：

- `--unshare-user`：用户命名空间隔离；
- `--unshare-pid`：PID namespace 隔离；
- `--unshare-ipc`：IPC namespace 隔离；
- `--unshare-net`：网络隔离，仅 `complete` 或显式 `networkMode=blocked` 时启用；
- `--ro-bind`：把 `/usr`、`/bin`、`/lib` 等系统目录只读挂载；
- `--bind`：只给临时目录可写；
- `/etc` 白名单挂载，避免直接暴露 `/etc/passwd`、`/etc/shadow` 等。

所以：

> 在 Linux / WSL2 且 bwrap 可用时，终端 sandbox 是 OS 级隔离，底层用到了内核 namespace 能力。

但它仍然不是 Firecracker microVM 或 gVisor 那种更强的虚拟化 / 用户态内核隔离。

### 4. macOS 上是不是内核级？

macOS 使用 `/usr/bin/sandbox-exec`，也叫 Seatbelt sandbox。

在 `packages/terminal-core/src/platform/darwin.ts` 中会生成 `.sb` profile，采用类似：

- `deny default`；
- 允许特定系统目录只读；
- project 目录只读；
- tmp 目录可读写；
- `filesystem` 级别允许网络；
- `complete` 级别不放开网络。

所以 macOS 也不是纯应用层 judge，而是可以使用系统原生 sandbox 能力限制子进程。

### 5. Windows 原生环境是什么情况？

Windows 原生不是内核级沙箱。

`packages/terminal-core/src/platform/windows.ts` 里写得很明确：

- WSL2 环境：委托给 LinuxSandbox，也就是 bwrap；
- Windows 原生：降级模式，不启用 OS 级沙箱；
- 安全性由 denylist + allowlist 保障；
- TODO 里提到后续可以接 Windows Sandbox API 或 AppContainer。

也就是说：

> 如果用户是在 Windows 原生 Electron/Daemon 里跑 Agent 终端命令，当前仓库实现更像“应用层 judge + 命令校验 + 路径边界”，不是内核级强隔离。

### 6. PTY 和 sandbox 的关系

还有一个容易误解的点：交互式 PTY 不一定能直接执行 sandbox 策略。

`packages/terminal-core/src/policy.ts` 里有逻辑：

- `route=blocked`：直接阻断，不能降级；
- `route=sandbox`：当前交互式 PTY 不支持时，可以降级到 `CommandExecutor`；
- `networkMode=blocked/custom`：PTY 不能保证网络限制时，也会降级到 `CommandExecutor`。

`packages/terminal-core/src/degraded-executor.ts` 的注释也说明：当交互式 PTY 不能满足 sandbox / network-restricted 策略时，会 fallback 到 `CommandExecutor`，也就是 spawn + OS sandbox 这条路径。

所以不是所有 terminal 都天然在 OS sandbox 里，关键要看：

- policy 是否是 `route=sandbox`；
- 是否有 `sandboxLevel`；
- 当前平台的 OS sandbox 是否可用；
- 是否从 PTY 降级到了 `CommandExecutor`。

### 7. 文件、Git、checkpoint 等 IPC 也靠应用层路径边界

除了 LLM tool call 的 judge，Electron 主进程还有一层路径访问检查，位置是：

- `apps/tabtin-electron/src/main/security/path-access-checker.ts`

它把文件系统 / git / checkpoint IPC 的路径边界收敛到同一套逻辑：

- 先查 hardline；
- 再查敏感路径；
- 再查 deny pattern；
- 再判断是否在 `WorkspaceSnapshot.allowedPaths` 或平台允许目录里；
- 不在 workspace 内就拒绝。

这层仍然是应用层边界，不是内核隔离。

### 8. 一句话总结

可以这样对外讲：

> TabTin 的安全沙箱不是单一机制，而是分层防护：上层用 security-policy 的 judge 做工具级、路径级、审批级决策；终端 sandbox 路径再尽量用 OS 能力隔离子进程。Linux/WSL2 用 bwrap，macOS 用 sandbox-exec；Windows 原生目前会降级为应用层 denylist/allowlist/路径边界。因此它不是纯应用层 judge，但也不是所有平台、所有命令都有内核级强沙箱。

如果再压缩成一句：

> **核心控制面是应用层 judge；终端执行面有可选 OS 级沙箱；Windows 原生会降级，不应宣传成全量内核级沙箱。**

## Q14：应用层 judge 的“三档审批”指的是什么？

这里的“三档审批”指 `ApprovalMode` 的三个取值：

- `always_ask`：请求批准
- `auto`：替我审批，也就是旧版本里的 yolo 语义
- `full_access`：完全访问

它不是 Agent 的工作模式。TabTin 里要区分两个维度：

| 维度 | 负责什么 | 例子 |
| --- | --- | --- |
| AgentMode | Agent 当前“做什么类型的事” | ask、plan、study、agent、group |
| ApprovalMode | Agent 做事时“多大程度需要问用户” | always_ask、auto、full_access |

代码里的单源在：

- `packages/agent-modes/src/types.ts`
- `packages/security-policy/src/build-policy.ts`
- `packages/security-policy/src/judge.ts`

`packages/agent-modes/src/types.ts` 里也明确说明：AgentMode 管工具集 / prompt / 工作方式，ApprovalMode 管 judge 审批档位。

### 1. `always_ask`：请求批准，默认最谨慎

`always_ask` 可以理解为：

> **默认比较谨慎，工作区内正常操作尽量不打扰；但一旦越界、敏感、破坏性或设备/MCP 类操作，就要问用户。**

典型行为：

- 工作区内普通读写：一般允许；
- 工作区外路径：ask；
- 敏感文件：ask 或 deny；
- 删除文件等破坏性操作：ask；
- MCP / device interact：通常 ask；
- 灾难级命令：deny。

适合场景：

- 默认安全模式；
- 新 Agent / 新项目首次使用；
- 用户希望每个越界动作都能看到确认弹窗；
- 团队协作里不希望 Agent 自作主张。

例子：

```text
Agent 想删除工作区内文件
        │
        ▼
judge 发现是 destructive operation
        │
        ▼
返回 ask：即将删除文件，请确认
```

### 2. `auto`：替我审批，减少打扰

`auto` 可以理解为：

> **用户已经授权 Agent 更主动地干活，大部分普通操作自动放行；但风险红线、工作区外敏感写等仍然不会完全无脑放行。**

它基本对应旧的 `yolo` 语义，但不是“完全没限制”。

代码里 `judge.ts` 的设计说明是：

- `auto` 会在 step 3 走 `auto_allow`，自动放行大多数操作；
- 但 `catastrophic` 灾难级红线三档都 deny；
- 部分风险红线在 `auto` 下不是直接 deny，而是转 ask；
- 工作区外敏感写等风险操作也可能转 ask。

典型行为：

- 工作区内普通读写：allow；
- 工作区外普通操作：大多 allow；
- 工作区内部分敏感 ask：可能被豁免，减少打扰；
- 风险命令，例如 sudo 类：ask；
- 灾难级命令，例如 `rm -rf /`：deny。

适合场景：

- 用户信任当前 Agent；
- 希望 Agent 连续执行任务，不要频繁弹窗；
- 但仍希望系统保留灾难级保护和高风险确认。

例子：

```text
Agent 想在项目外创建一个普通临时文件
        │
        ▼
approvalMode = auto
        │
        ▼
judge 返回 allow / auto_allow
```

但如果是：

```text
Agent 想执行 sudo 或高风险系统命令
        │
        ▼
approvalMode = auto
        │
        ▼
judge 仍可能返回 ask
```

### 3. `full_access`：完全访问，最大放权

`full_access` 可以理解为：

> **最大程度放权，除了灾难级命令仍然禁止，其余尽量放行。**

它比 `auto` 更宽松。

典型行为：

- 工作区内操作：allow；
- 工作区外操作：allow；
- 普通敏感 ask：一般绕过；
- 风险操作：更倾向放行；
- 灾难级 hardline：仍然 deny。

也就是说，`full_access` 不是“关闭安全系统”，而是把审批打扰降到最低，只保留最底线红线。

适合场景：

- 高信任单人本地环境；
- 用户明确知道风险；
- 需要 Agent 大范围操作本机文件或执行复杂自动化任务。

不适合场景：

- 群协作 Space；
- 不熟悉的仓库；
- 需要严格审计的生产环境；
- 执行来源不可信的命令。

### 4. 三档审批和 group Space 的关系

`packages/security-policy/src/build-policy.ts` 里有一个重要规则：

> **群协作 Space 与 `auto` / `full_access` 互斥。**

也就是说，如果是 group space，最终派生出来的审批档会被钉到 `always_ask`。

原因很好理解：

- 单人本地空间里，用户可以对自己的机器承担风险；
- 群协作空间里，Agent 的动作可能影响多人共享上下文、共享资源或团队数据；
- 所以不能让某个人的“全自动授权”直接覆盖团队空间的安全边界。

### 5. 三档审批和 `approval_grant` 的关系

`approval_grant` 是 Agent 配置里持久化的授权上限，位于：

```text
AgentConfigV3.security.approval_grant
```

取值也是：

```text
always_ask / auto / full_access
```

`build-policy.ts` 会从 AgentConfig 派生本轮真正生效的 `approvalMode`。

规则大致是：

```text
AgentConfig.security.approval_grant
        │
        ▼
buildPolicyFromAgentConfigV2
        │
        ▼
EffectivePolicy.approvalMode
        │
        ▼
judge.ts 按该档位做 allow / ask / deny
```

还有一个兼容逻辑：旧字段 `allow_yolo_mode=true` 会被读成 `auto`。

### 6. 三档审批的差异速记表

| 审批档 | 中文理解 | 普通工作区操作 | 工作区外普通操作 | 敏感 / 高风险操作 | 灾难级红线 |
| --- | --- | --- | --- | --- | --- |
| `always_ask` | 请求批准 | 多数 allow | ask | ask / deny | deny |
| `auto` | 替我审批 | allow | 多数 allow | 部分 ask，部分 allow | deny |
| `full_access` | 完全访问 | allow | allow | 多数 allow | deny |

### 7. 对外分享时怎么讲？

可以这样讲：

> TabTin 的应用层 judge 有三档审批：`always_ask`、`auto`、`full_access`。它们控制的不是 Agent 会不会思考，而是 Agent 在调用工具、读写文件、执行命令、操作设备时，到底要不要先问用户。`always_ask` 最谨慎，越界和风险动作都问；`auto` 是替我审批，大部分自动放行但高风险仍问；`full_access` 是最大放权，只保留灾难级红线。

再压缩成一句：

> **三档审批就是 TabTin 对 Agent 工具执行的放权程度：默认问、替我批、完全放权。**

---

## Q15：共享事件流（CRDT 同步 + Agent 事件）是什么意思？CRDT 不是只同步元数据吗？

### 1. 先给结论

这里的“共享事件流”不是说 **CRDT 和 Agent 事件是同一种数据结构**，而是说：

> TabTin 把协作空间里需要多端实时感知的变化，统一抽象成一组可订阅的实时流。里面一类是 CRDT/Y.js 负责的共享状态同步，另一类是 Agent Runtime 产生的过程事件流。

所以可以拆成两层理解：

```text
共享事件流 = 协作空间内的实时变化总线
          = CRDT 协同状态变化
          + Agent 执行过程事件
          + 资源/通知/设备/审批等业务事件
```

一句话：

> **CRDT 负责“共享状态怎么合并成一致”，Agent 事件负责“执行过程怎么实时广播”，共享事件流是把这些实时变化按 space/session/table 等作用域分发给订阅者的架构说法。**

### 2. CRDT 不是只同步 metadata

CRDT 不是专门用来同步元数据的。它的本质是：

> 用于多人、多端、离线/并发编辑场景下，把同一份共享状态自动合并到一致结果的数据结构/算法。

metadata 可以放进 CRDT 文档里同步，但 metadata 只是其中一部分。

在 TabTin 项目里，CRDT/Y.js 同步的范围明显比 metadata 大得多。以 TabData 为例，`packages/table-engine/src/collab/ydoc-schema.ts` 和 `apps/collab-live/src/extensions/table-database.ts` 里定义的 Y.Doc 结构包括：

```text
records      表格行数据：Y.Map<recordId, Y.Map<fieldId, cellValue>>
rowOrderMap  行顺序投影
views        视图定义：筛选、排序、分组、列配置等
meta        字段、版本、表名、表 ID 等元信息
```

也就是说，CRDT 既同步：

- 表格记录/单元格值；
- 行顺序；
- 视图配置；
- 字段 schema；
- 文档正文的 Y.js binary；
- canvas / slide / video 等协作状态；

也同步 metadata，但不止 metadata。

### 3. 项目里的 CRDT 同步链路是什么？

项目里有一个独立的协作实时服务：

```text
apps/collab-live/
```

`apps/collab-live/src/server.ts` 明确说明它是：

```text
Express + Hocuspocus WebSocket 服务器
WebSocket: Y.js 实时协作
HTTP API: 格式转换端点、Agent push、Block 操作（供 Django 调用）
```

它注册了多条 Y.js 协作 WebSocket 端点：

```text
/collaboration          TabDoc 文档协作
/table-collaboration    TabData 表格协作
/slide-collaboration    Slide 协作
/video-collaboration    Video 协作
/canvas-collaboration   Canvas 协作
```

典型链路是：

```text
多人/多端编辑文档或表格
        │
        ▼
collab-live / Hocuspocus / Y.js
        │
        ▼
Y.Doc 接收 update，并由 CRDT 合并并发修改
        │
        ├─ 在线客户端实时看到共享状态变化
        └─ debounce 后写回 Django / PostgreSQL 持久化
```

对表格来说，`TableDatabase.buildPersistPayload()` 会从 Y.Doc 里计算：

- `changed_records`
- `new_records`
- `deleted_record_ids`
- `row_order`
- `fields`
- `views`
- `base_version`

再写回 Django。这个过程说明 Y.Doc 不是只保存“表格元信息”，而是保存表格协作状态本身。

### 4. Agent 事件流同步的是什么？

Agent 事件流不是 CRDT。它同步的是 Agent 执行过程中的“直播事件”。

项目的实时契约在：

```text
contracts/asyncapi/ws-gateway.yaml
contracts/models/shared-models.yaml
packages/agent-wire/src/events.ts
packages/ws-gateway-client/src/events.ts
```

里面定义了大量 `agent.stream.*` 事件，例如：

```text
agent.stream.lifecycle              运行生命周期变化
agent.stream.message_start          一轮消息开始
agent.stream.content_block_delta    内容块增量，例如模型输出文本/工具参数增量
agent.stream.message_stop           一轮消息结束
agent.stream.step                   步骤更新
agent.stream.done                   流结束
agent.stream.message_persisted      消息已持久化
agent.stream.todo                   Todo 列表更新
agent.stream.ssh_output             SSH/终端输出
agent.stream.compaction             上下文压缩通知
agent.stream.ask_user_required      需要用户回答
agent.stream.request_approval_required 需要用户审批
```

这些事件的目的不是“冲突合并”，而是让 Electron/Web/Mobile 等客户端实时看到：

- Agent 当前跑到哪一步；
- 模型正在输出什么；
- 工具调用开始/结束/失败；
- 是否需要用户审批；
- 子 Agent 是否启动/完成；
- 消息是否已经落库；
- todo、上下文压缩、终端输出等状态变化。

典型链路是：

```text
Agent Runtime 执行任务
        │
        ▼
产生 agent.stream.* 事件
        │
        ▼
Django / WS Gateway 发布到 topic
        │
        ▼
订阅该 session/thread 的客户端实时渲染
```

### 5. 那“共享事件流”到底是什么意思？

可以把它理解成产品/架构层的统一说法，而不是某一个具体库名。

在一个协作 Space 或一个 Agent Session 中，用户关心的不只是“最终数据是什么”，还关心“过程中发生了什么”。因此项目把不同来源的实时变化都按 topic 分发：

- 文档/表格/画布状态变化：主要走 Y.js/CRDT 协作链路；
- Agent 执行过程：走 `agent.stream.*`；
- Space 资源列表变化：走 `context.sync.*`；
- 表格兼容事件：走 `table.events.*`；
- 通知、设备、Git 状态、审批等：走各自 topic。

从使用体验上看，大家订阅的是“同一个协作上下文里的实时变化”；从实现上看，它们可能来自不同通道、不同协议。

### 6. CRDT 和 WS Gateway 是不是同一条通道？

不是完全同一条。

更准确地说，TabTin 里至少有两类实时链路：

#### A. CRDT/Y.js 协作链路

由 `apps/collab-live` 提供，基于 Hocuspocus/Y.js：

```text
/collaboration
/table-collaboration
/slide-collaboration
/video-collaboration
/canvas-collaboration
```

它解决的问题是：

> 多人同时编辑同一份文档/表格/画布时，状态如何合并、同步、持久化。

#### B. WebSocket Gateway 业务事件链路

由 AsyncAPI 契约定义：

```text
/ws/v1/gateway
```

它解决的问题是：

> Agent 流、通知、审批、设备状态、资源刷新、兼容表格事件等，如何统一按 topic 推给客户端。

`contracts/asyncapi/ws-gateway.yaml` 里的 `x-topic-capabilities` 包含：

```text
context.sync
agent.stream
agent.action
table.events
trace.stream
llm.stream
docparse.events
git.status
notifications
extension.events
```

所以：

```text
CRDT/Y.js 链路：偏“共享状态同步”
WS Gateway：偏“业务事件分发/订阅/重放”
```

上层产品可能把两者都叫作“共享事件流”的组成部分，但底层技术职责不同。

### 7. `table.events.*` 和 CRDT 的关系

这里尤其容易混淆。

在 AsyncAPI 中确实有：

```text
table.events.delta
table.events.field
table.events.view
```

但 `apps/tabtin_django/apps/tabdata/services/table_event_service.py` 的注释已经说明：

```text
主链路：collab-live stateless broadcast（Y.js 协作实例通知）
旧链路（已废弃）：WS Gateway 事件（table.events.delta/field/view）
Y.js 协作已成为 TabData 默认链路。WS Gateway 事件仅保留向后兼容。
```

也就是说，对于 TabData 表格：

- 现在主同步链路是 Y.Doc/CRDT；
- `table.events.delta/field/view` 更像旧版或兼容用的业务通知；
- 有些情况下 Django 仍会通过 collab-live 的 stateless broadcast 发轻量事件，例如评论变化、前端本地回声抑制等；
- 不应把 `table.events.delta` 等同于完整 CRDT update。

### 8. 三类事件的区别

| 类型 | 同步内容 | 是否需要冲突合并 | 最终权威状态在哪里 | 例子 |
| --- | --- | --- | --- | --- |
| CRDT/Y.js 同步 | 共享文档/表格/画布状态 | 需要，CRDT 自动合并 | Y.Doc 合并后写回 Django/PostgreSQL | 文档正文、表格 cells、行顺序、fields、views |
| Agent stream | Agent 执行过程直播 | 通常不需要 CRDT 合并 | 部分落 ChatMessage/日志，部分只是实时通知 | message_start、content_block_delta、tool、todo、approval_required、done |
| context/table/notification 业务事件 | 缓存失效、资源刷新、状态提醒 | 通常不需要 | 以业务表/API 查询结果为准 | context.sync.resource_updated、table.events.field、notifications |

### 9. 对外分享时怎么讲？

可以这样讲：

> TabTin 的“共享事件流”不是单指 CRDT，也不是单指 Agent 输出流，而是一个协作空间内的实时变化总线。CRDT/Y.js 负责把多人可编辑的共享状态合并成一致结果；Agent stream 负责把 Agent 的执行过程、模型输出、工具调用、审批请求实时广播给各端；context.sync/table.events/notifications 则负责资源刷新和业务通知。CRDT 可以同步 metadata，但它并不只同步 metadata，在 TabTin 里它还同步文档正文、表格记录、字段、视图、行顺序等共享状态。

压缩成一句：

> **CRDT 管“最终共享状态”，Agent stream 管“执行过程直播”，共享事件流管“把这些变化实时送给该看到的人”。**

