# ⭐游戏平台总览⭐

## 1. Overview

**统一身份平台（MP） + 可复制的游戏后端模板（GameTemplate） + 每个游戏独立部署，配套可复用组件（Mail / BugReport）与公司级管理后台（Dashboard）。**

核心原则：

- **代码 / 镜像可复用**
- **运行时与数据完全独立**
- **平台只做身份与治理，不做游戏业务**

---

## 2. 仓库地图

### 平台核心（只部署一份）

| 仓库 | 职责 | 状态 |
|------|------|------|
| [MP](https://github.com/setsuodu/MP) | 统一账号、JWT、渠道登录、游戏 Catalog | ✅ 已跑通 |
| [GameDashboard](https://github.com/setsuodu/GameDashboard) | 公司级管理后台（GM / 运营 / 程序） | 🚧 已有仓库，待完善 |

### 游戏生产流水线

| 仓库 | 职责 | 状态 |
|------|------|------|
| [GameTemplate](https://github.com/setsuodu/GameTemplate) | 游戏后端标准模板（建议设为 Public Template） | ✅ 已可用 |
| [game-match3-server](https://github.com/setsuodu/game-match3-server) | 三消游戏后端（首个落地实例） | ✅ 已跑通 |
| [game-match3-client](https://github.com/setsuodu/game-match3-client) | 三消 Unity 客户端 | 🚧 联调已通，客户端完善中 |

### 可复用组件（像 nginx 一样提供镜像）

| 仓库 | 职责 | 状态 |
|------|------|------|
| [AssetBundleFramework](https://github.com/setsuodu/AssetBundleFramework) | AssetBundle 框架 OpenUPM | 🚧 已有仓库 |
| [ExcelConfigCompiler](https://github.com/setsuodu/ExcelConfigCompiler) | Excel 导表工具 OpenUPM | 🚧 已有仓库 |
| [BugReport](https://github.com/setsuodu/BugReport) | 异常/反馈收集服务 + 后续客户端 SDK | 🚧 已有仓库 |
| Mail（待建） | 游戏内邮件 / 补偿 | 📋 规划中 |

---

## 3. 架构全景

```text
                        ┌─────────────────────────────────┐
                        │              MP                 │
                        │  统一身份 / JWT / Catalog       │
                        │  （全公司只部署一份）            │
                        └────────────────┬────────────────┘
                                         │ Bearer JWT
           ┌─────────────────────────────┼─────────────────────────────┐
           │                             │                             │
           ▼                             ▼                             ▼
   ┌───────────────┐             ┌───────────────┐             ┌───────────────┐
   │ game-match3   │             │ game-xxx      │             │ game-yyy      │
   │ 独立 Repo     │             │ 独立 Repo      │             │ 独立 Repo     │
   │ 独立 Image    │             │ 独立 Image     │             │ 独立 Image    │
   │ 独立 DB       │             │ 独立 DB        │             │ 独立 DB       │
   │ 独立发布       │             │ 独立发布       │             │ 独立发布      │
   └───────┬───────┘             └───────┬───────┘             └───────┬───────┘
           │                             │                             │
           │  按需引用组件镜像             │                             │
           ▼                             ▼                             ▼
   ┌───────────────┐             ┌───────────────┐             ┌───────────────┐
   │ BugReport     │             │ Mail          │             │ ...           │
   │ (独立镜像)     │             │ (独立镜像)     │             │               │
   └───────────────┘             └───────────────┘             └───────────────┘

                        ┌─────────────────────────────────┐
                        │         GameDashboard           │
                        │  公司级后台（只部署一份）         │
                        │  GM 封号 / 运营发奖 / 程序查 Bug │
                        │  按角色 + 游戏权限隔离           │
                        └─────────────────────────────────┘
```

### 关键边界（必须遵守）

| 层级 | 做什么 | 仓库 |
| ---- | ---- | ---- |
| **中台** | 登录、JWT、游戏注册表 | https://github.com/LongLongGames/MP |
| **模板** | 提供可复制的后端骨架 | https://github.com/LongLongGames/GameTemplate |
| **游戏** | 玩法、存档、排行榜、版本、资源 | https://github.com/LongLongGames/game-match3-server<br>https://github.com/LongLongGames/game-match3-client |
| **组件** | 提供标准化镜像 | https://github.com/LongLongGames/BugReport<br>https://github.com/LongLongGames/Mail |
| **后台** | 聚合管理、权限控制 | https://github.com/LongLongGames/GameDashboard |

---

## 4. 技术统一约定

- 语言 / 运行时：**.NET 10 + Native AOT**
- 数据库：PostgreSQL 16（每游戏独立实例或独立库）
- 缓存：Redis 7
- 网关：Nginx
- 鉴权：统一 JWT（HS256，MP 与游戏共用 Secret）
- 迁移：DbUp（嵌入式 SQL）——**必须通过 `--migrate` 专用入口执行**，禁止在多副本 API 启动路径中直接跑迁移。标准做法见 [GameTemplate](https://github.com/LongLongGames/GameTemplate)（单镜像 + Compose 一次性 Job + `service_completed_successfully`）
- 交付：Docker + GitHub Actions + GHCR
- 客户端：Unity（首个已验证）

---

## 5. 宿主机端口规划

容器内仍使用标准端口；**仅宿主机映射**按本表执行，避免与经典服务及兄弟业务冲突。

### 分段

| 分区 | 段 | 说明 |
|------|-----|------|
| 平台 MP / Dashboard | 11000–11999 | 全公司一份 |
| 可复用组件 | 12000–12999 | BugReport / Mail / … |
| 游戏实例 | 13000+ | 每游戏一块 100 端口 |

### 平台

| 服务 | Host | Container |
|------|------|-----------|
| mp-gateway | 11080 | 80 |
| mp-postgres | 11032 | 5432 |
| GameDashboard | 11090 | 80 |

### 组件

| 服务 | Host | Container |
|------|------|-----------|
| bugreport-server | 12080 | 8080 |
| bugreport-postgres | 12032 | 5432 |
| bugreport-minio S3 | 12090 | 9000 |
| bugreport-minio Console | 12091 | 9001 |
| mail-server | 12180 | 8080 |
| mail-postgres | 12132 | 5432 |

### 游戏（G = 1,2,3…）

| 角色 | 公式 | match3 (G=1) |
|------|------|----------------|
| Gateway | 13000 + G×100 + 80 | 13180 |
| Postgres | 13000 + G×100 + 32 | 13132 |
| Redis | 13000 + G×100 + 79 | 13179 |

> 内部 user / core / leaderboard 服务**不映射**宿主机端口，只通过本游戏 Gateway 访问。

---

## 6. 当前已验证能力（2026-08）

- MP 登录（official / guest）并签发 JWT
- Match3 后端独立部署（User / Leaderboard / Core / Gateway）
- Unity 客户端完成：登录 → 拉资料 → 提交分数 → 查排行榜 → 查自己排名
- 多服务 Docker Compose 编排稳定运行
- 游戏与平台 JWT 联调通过

---

## 7. Roadmap（公司级）

[Projects V2](https://github.com/orgs/LongLongGames/projects/1)

---

## 8. 新游戏接入 SOP（最短路径）

1. 在 **MP Catalog** 注册 game_id
2. 从 **GameTemplate** 使用 “Use this template” 创建新仓库
3. 修改 GAME_ID、玩法逻辑、表结构
4. 配置与 MP 相同的 JWT_SECRET
5. docker compose up 验证
6. 按需在 compose 中引用 bugreport / mail 镜像
7. 客户端先调 MP 拿 Token，再调本游戏 Gateway

详细步骤见各仓库 README。

## 9. 工程规范（ADR）

影响全组织服务的架构决策（正文在 [.github/docs/adr](https://github.com/LongLongGames/.github/tree/main/docs/adr)）：

| ADR | 说明 |
|-----|------|
| [ADR-0001](https://github.com/LongLongGames/.github/blob/main/docs/adr/0001-dbup-migrate-job.md) | DbUp：迁移与 API 进程分离 |
| [ADR-0002](https://github.com/LongLongGames/.github/blob/main/docs/adr/0002-aot-json-and-jwt.md) | AOT：禁止匿名错误 JSON；JWT 对齐 MP SimpleJwt；API smoke 含 bad path |
