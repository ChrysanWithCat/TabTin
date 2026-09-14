# TabTin 本地启动说明

> 适用仓库：`D:\1-github\TabTin`（Community 版源码）。
> 本文描述当前机器上验证过的启动方式：**服务端全部放进 Podman 容器**，**桌面端与 AdminDash 管理端在本机（Windows）运行**，同时补充仓库中**未启动模块**的用途，便于了解完整系统。

## 1. 仓库全部模块说明

### 1.1 运行模块一览（apps/ 下与基础服务）

| 模块 | 仓库位置 | 运行位置 | 端口 / 入口 | 作用 | 本说明状态 |
| --- | --- | --- | --- | --- | --- |
| Django API 服务端 | `apps/tabtin_django` | Podman 容器 `tabtin-community-django-1` | `127.0.0.1:6060` | 核心后端。账号注册/登录、组织与 Space、表格、文档、会话、模型配置、文件上传、运营管理 API 与 `/health`、`/health/ready` 健康检查。 | 已启动 |
| Celery 后台任务 | `apps/tabtin_django` | Podman 容器 `tabtin-community-celery-1` | 无对外端口（经 Redis broker） | 异步与定时任务：文档解析/导出、PDF/PPT、向量索引与 RAG、通知推送、运营统计、钱包对账、定时清理等。 | 已启动 |
| Centrifugo 实时消息 | 仓库外独立服务 | Podman 容器 `tabtin-community-centrifugo-1` | `127.0.0.1:8100` | 实时通道 / WebSocket 消息服务，支撑聊天消息、通知与页面实时刷新；健康检查 `/health`。 | 已启动 |
| PostgreSQL | 仓库外基础服务 | Podman 容器 `tabtin-community-postgres-1` | 仅容器内网 | 业务主数据库（账号、组织、表格、文档、消息等），启用 pgvector 扩展。 | 已启动 |
| Redis | 仓库外基础服务 | Podman 容器 `tabtin-community-redis-1` | 仅容器内网 | 缓存、Celery broker、Channels/Centrifugo 依赖。 | 已启动 |
| Electron 桌面端 | `apps/tabtin-electron` | 本机 Node/Electron | 渲染页 `127.0.0.1:5175` | TabTin 桌面客户端（Agent 工作平台），联调时由 electron-vite 启动，自动连接 `6060` API 与 `8100` 实时通道。 | 已启动 |
| AdminDash 管理端 | `apps/admindash` | 本机 Node/Vite | `127.0.0.1:5174` | 运营/管理后台 Web 界面（模型、账号、组织、表格、文档、OSS、账单等），本地 Vite 把 `/api` 等请求代理到 `6060`。 | 已启动 |
| Collab Live 实时协作服务 | `apps/collab-live` | 本机 Node（官方流程，未容器化） | `127.0.0.1:4100` | 统一实时协作服务（Hocuspocus / Y.js WebSocket Server），负责文档、表格、演示文稿的多人实时协同编辑。 | 可选启动（见 3.5） |
| tabtin-web 在线平台 | `apps/tabtin-web` | 本机 Node/Vite | `127.0.0.1:5176` | 云上桌面 / 在线平台入口，也是公开分享页面的宿主；全量预览会启动它。本方案不启动。 | 未启动 |
| tabtin-daemon Agent 守护进程 | `apps/tabtin-daemon` | 独立进程（本地或远程无头机） | 由 `start` 子命令启动 | Agent Daemon：无界面执行运行时，供远程服务器 / 无人值守场景承载 Agent 任务与设备控制。桌面端可管理并与之协作。 | 未启动 |
| TabTin Android 客户端 | `apps/tabtin-android` | Android Studio / Gradle | Debug APK | 移动端配套客户端，用于在手机上查看、发起或控制桌面 Agent 任务（不独立执行 Agent）。 | 未启动 |
| TabTin iOS 客户端 | `apps/tabtin-ios` | Xcode | iOS Debug 包 | 与 Android 客户端同属移动配套入口，需桌面端完成设备绑定后使用。 | 未启动 |

> “未启动”指本机当前这套“服务端容器 + 两个本地界面”的运行方案没有拉起它们，不代表仓库不包含这些模块。需要时按各自 README 单独启动/构建即可。

### 1.2 Django 服务端内部功能域

`apps/tabtin_django` 是单体服务端，API 能力按功能域组织；下表是按业务域归纳的说明（不逐一列出每个代码目录）：

| 功能域 | 说明 |
| --- | --- |
| 用户、登录与后台权限 | 账号注册/登录、手机号/邮箱登录、会话、设备绑定、系统角色与后台 RBAC、操作审计。 |
| 组织与空间 | Organization / Workspace / Agent / Device 模型，工作区、项目、分享、成员邀请与权限控制。 |
| Agent 引擎与任务 | Agent 运行、任务/会话编排、子任务、执行轨迹、工具/能力注册与权限审计。 |
| 技能（Skills） | 技能包目录、启停与偏好、技能与 Agent 绑定。 |
| 表格 | 数据表、字段类型、视图、记录历史、导入导出、公式/按钮、API Token、Webhook。 |
| 文档 | 富文本文档、版本历史、文档分享、评论与附件。 |
| 演示文稿 | PPT/Slides 工程、页面与元素变更、分享与导入导出。 |
| 便签、记忆与画像 | 便签/日记、Agent 长期记忆、用户画像与特征蒸馏。 |
| 实时协作 | 文档/表格/演示的 Y.js 协同后端接口与适配器（对接到 Collab Live）。 |
| IM 与消息 | 会话/群组、消息、提及、通知中心与事件总线。 |
| 模型与 LLM | Provider/模型目录、场景绑定、模型调用、Byok、成本计量、LiteLLM 网关适配。 |
| 计费与钱包 | 用量统计、钱包、账单、定价与运营计量。 |
| RAG / 搜索 / Embedding | 向量库、文档/表格/代码检索、FTS 全文搜索与统一搜索。 |
| 文件与 OSS | 文件上传/下载、本地 OSS、媒体资源管理。 |
| 第三方集成 | Webhook 出站、GitHub、飞书、短信/邮件、支付/云厂商 SDK 适配。 |
| 运营与诊断 | 客户端错误上报、诊断包、版本更新策略、平台配置、维护任务。 |

### 1.3 共享库包（packages/*）

`packages/*` 是供上述应用复用的 TypeScript/前端/协议库（如 `contracts`、`api-client`、`table-*`、`doc-*`、`app-shell`、`agent-*`、`browser-*` 等），没有独立启动入口，作为 workspace 依赖被各应用构建/引用，本文不逐一展开。

### 1.4 本方案的运行结构

```text
┌─────────────────────────── 本机（Windows） ───────────────────────────┐
│  Electron 桌面端（TabTin Dev）            AdminDash 管理端（Vite）      │
│  http://127.0.0.1:5175                    http://127.0.0.1:5174        │
└──────────────────────────────┬────────────────────────────────────────┘
                               │ HTTP / WS
┌──────────────────────────────┴────────── Podman 容器 ─────────────────┐
│  Django  API :6060        Centrifugo 实时 :8100                       │
│  Celery 后台任务            Redis / PostgreSQL                        │
└───────────────────────────────────────────────────────────────────────┘
```

## 2. 前置条件

- Windows 上已安装：Node.js（仓库要求 `>=18`）、pnpm `9.15.0`、Podman（含已初始化的 `podman-machine-default`）。
- 仓库依赖已安装：`pnpm install --frozen-lockfile`（本机 `node_modules` 存在）。
- 根目录存在 `.env`、`.env.community-runtime`（Compose 读取）。
- 本说明不要求 Docker Desktop；`docker` 命令通过 Podman 的 Docker 兼容 API 工作。

## 3. 启动步骤

所有命令在仓库根目录的 **PowerShell** 中执行。

### 3.1 启动 Podman 容器引擎

```powershell
podman machine start
```

Podman 启动后会把 Docker 兼容 API 转发到 `npipe:////./pipe/docker_engine`。为了让仓库里的 `docker compose` 脚本指向 Podman，每次新开终端先设置：

```powershell
$env:DOCKER_HOST = 'npipe:////./pipe/docker_engine'
```

### 3.2 启动服务端（全部进 Podman）

```powershell
docker compose -f compose.yaml -f compose.community-dev.yaml -f .celery-health.override.yaml up -d
```

说明：

- `compose.yaml`：社区服务端基础定义（Django / Celery / Centrifugo / Postgres / Redis）。
- `compose.community-dev.yaml`：本地开发叠加层，把后端源码挂载进容器，便于开发时跟随改动。
- `.celery-health.override.yaml`：**Podman 下必须带**。Podman 容器没有 `HOSTNAME` 环境变量，官方 Celery 健康检查 `-d celery@$HOSTNAME` 会永远匹配不到节点；该 override 改为不带目标节点的 ping，并把超时放宽。
- 首次启动若本机没有 `tabtin/community-django:dev` 镜像，Compose 会自动构建（耗时较长）；后续为增量热更可不重建。

等待健康：

```powershell
docker compose -f compose.yaml -f compose.community-dev.yaml -f .celery-health.override.yaml ps
```

期望 Django / Celery / Centrifugo / Postgres / Redis 全部为 `healthy` 或 `Up`。

### 3.3 启动 AdminDash 管理端（本机）

另开一个 PowerShell，在仓库根目录执行：

```powershell
pnpm --filter admindash dev
```

- 首次执行会先构建约 18 个 workspace 依赖包，耗时可能超过 1 分钟。
- 启动成功后访问：<http://127.0.0.1:5174>
- `Ctrl+C` 停止。

### 3.4 启动 Electron 桌面端（本机）

再开一个 PowerShell，在仓库根目录执行：

```powershell
pnpm --filter tabtin-electron dev
```

- 首次执行会做 Electron workspace 预构建（73 个依赖包）与可选运行时检查，随后弹出 `TabTin Dev` 窗口。
- 渲染开发服务在：<http://127.0.0.1:5175>
- `Ctrl+C` 停止。

> 提示：也可以使用 `node scripts/dev.mjs electron`，但注意 `node scripts/dev.mjs admindash` 内部只等 60 秒健康检查，首次构建较慢时父进程会先报失败（Vite 子进程通常会继续运行）；需要稳定前台管理时建议直接使用上面的 `pnpm --filter ... dev`。

### 3.5 启动 Collab Live 实时协作服务（可选，本机）

只有需要验证 **TabDoc / TabData / TabSlide 等多人实时协同编辑** 时才需要启动 Collab Live；普通登录、管理后台、IM、Agent 会话等功能不依赖它。

启动前先确认 3.2 的 Django 服务端已经健康，因为 Collab Live 会通过 `DJANGO_API_URL` 调用 Django 做协作鉴权和数据读写。

推荐使用仓库提供的 Windows 脚本启动：

```powershell
scripts\backend\collab-live-start.bat
```

该脚本会自动完成以下事情：

- 读取仓库根目录 `.env` 与 `scripts\backend\_dev-env.bat` 中的本地开发端口配置；默认 `DJANGO_BIND_PORT=6060`、`COLLAB_LIVE_PORT=4100`。
- 预构建 `collab-live` 所需 workspace 依赖。
- 清理已占用的 `4100` 端口。
- 设置 `NODE_ENV=development`、`PORT=4100`、`DJANGO_API_URL=http://127.0.0.1:6060`。
- 在后台启动 `apps/collab-live`，实际执行的是 `pnpm exec tsx src/start.ts`。
- 日志写入 `apps\tabtin_django\logs\collab-live.log` 和 `apps\tabtin_django\logs\collab-live.error.log`，PID 写入 `apps\tabtin_django\logs\collab-live.pid`。

启动后检查健康：

```powershell
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:4100/health
```

期望返回：

```json
{"status":"ok"}
```

Collab Live 对外提供的本地 WebSocket 端点如下：

| 类型 | 地址 |
| --- | --- |
| 文档协作 | `ws://127.0.0.1:4100/collaboration` |
| 表格协作 | `ws://127.0.0.1:4100/table-collaboration` |
| 演示文稿协作 | `ws://127.0.0.1:4100/slide-collaboration` |
| 视频协作 | `ws://127.0.0.1:4100/video-collaboration` |
| 画布协作 | `ws://127.0.0.1:4100/canvas-collaboration` |

Electron 渲染端默认会从 `apps/tabtin-electron/src/renderer/src/config/api.ts` 使用 `ws://localhost:4100` 拼出这些协作地址；如果需要局域网设备访问，可在 Electron 的本地环境文件中显式配置 `VITE_COLLAB_WS_BASE=ws://<YOUR_LAN_IP>:4100`。

如需前台调试 Collab Live，也可以另开 PowerShell 手动运行：

```powershell
$env:NODE_ENV = 'development'
$env:PORT = '4100'
$env:DJANGO_API_URL = 'http://127.0.0.1:6060'
pnpm --filter collab-live start
```

> 注意：`apps/collab-live/package.json` 里的 `dev` 脚本使用 `NODE_ENV=development tsx watch ...` 这种 POSIX 写法，在 Windows PowerShell 下不如上面的 `.bat` 脚本稳定；本机 Windows 推荐优先使用 `scripts\backend\collab-live-start.bat`。

## 4. 验证清单

启动完成后逐项确认：

| 项目 | 地址 | 期望 |
| --- | --- | --- |
| 管理端 | <http://127.0.0.1:5174> | 页面 200，可登录/注册 |
| 桌面端渲染页 | <http://127.0.0.1:5175> | 页面 200 |
| Django 健康 | <http://127.0.0.1:6060/health> | 200 |
| Django 就绪 | <http://127.0.0.1:6060/health/ready> | 200 |
| Centrifugo 健康 | <http://127.0.0.1:8100/health> | 200 |
| Collab Live 健康（可选） | <http://127.0.0.1:4100/health> | 200，返回 `{"status":"ok"}` |
| 桌面端窗口 | 本机 | `TabTin - AI 工作平台` 窗口已打开 |

PowerShell 一键检查：

```powershell
foreach ($u in @('http://127.0.0.1:5174/','http://127.0.0.1:5175/','http://127.0.0.1:6060/health','http://127.0.0.1:6060/health/ready','http://127.0.0.1:8100/health','http://127.0.0.1:4100/health')) {
  try { (Invoke-WebRequest -UseBasicParsing -TimeoutSec 5 $u).StatusCode } catch { "FAIL $u" }
}
```

> 如果没有执行 3.5 启动 Collab Live，`http://127.0.0.1:4100/health` 显示 `FAIL` 是正常的。

## 5. 停止

### 停止本机前端

在运行 AdminDash / Electron 的 PowerShell 里分别按 `Ctrl+C`。如果 Collab Live 是用前台方式启动的，也在对应 PowerShell 里按 `Ctrl+C`。

### 停止 Collab Live（可选）

如果使用 `scripts\backend\collab-live-start.bat` 后台启动，可在仓库根目录执行：

```powershell
scripts\backend\collab-live-stop.bat
```

该脚本会按 `COLLAB_LIVE_PORT` 清理本机 `4100` 端口上的 Collab Live 进程。

### 停止 Podman 服务端

```powershell
$env:DOCKER_HOST = 'npipe:////./pipe/docker_engine'
docker compose -f compose.yaml -f compose.community-dev.yaml -f .celery-health.override.yaml down
```

- `down` 不删除数据卷，容器数据（PostgreSQL/Redis 等）会保留。
- 不要轻易使用 `down -v`，那会清空本地业务数据库。
- 如需同时关闭 Podman 虚拟机：`podman machine stop`。

## 6. 注意事项

1. **Celery 健康检查**：必须带 `.celery-health.override.yaml` 启动，否则 Celery 容器在 Podman 下会一直显示 `unhealthy`（原因见 3.2）。
2. **Collab Live**：仓库没有 Collab 的容器化定义。需要文档/表格/演示文稿实时协作时，按 3.5 使用官方本地脚本 `scripts\backend\collab-live-start.bat` 单独启动（运行在 `4100`）；不需要实时协作时可以不启动。
3. **Python**：本说明的服务端运行在容器内（Python 3.11），不依赖本机 Python；本机只需 Node/pnpm。
4. **数据持久化**：账号与业务数据在 Podman 数据卷中，重新 `up -d` 不会丢数据。
5. **首次构建较慢**：workspace 预构建、Django 镜像构建或首次容器迁移都需要数分钟，属正常现象。
