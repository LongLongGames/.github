# ADR-0001：数据库迁移与 API 进程分离（DbUp）

- **状态：** Accepted  
- **日期：** 2026-09-12  
- **影响面：** 所有使用 PostgreSQL + DbUp 的 .NET 服务（MP、Mail、BugReport、Game.* 等）  
- **相关：** GameTemplate、compose 启动顺序、GHCR 镜像入口

## 背景

早期部分服务在 **API 进程启动时** 同步执行 DbUp。多副本 / 滚动发布时会出现：

- 多个实例同时跑迁移 → 锁竞争、失败重试、启动变慢
- 迁移失败与进程存活耦合 → 健康检查与发布语义混乱
- 本地与生产「谁负责 schema」不清晰

## 决策

1. **迁移与 API 进程分离**  
   - API 容器 **不** 在启动路径执行 DbUp  
   - 使用 **一次性 Job**（同镜像、不同入口参数）执行迁移  

2. **同一镜像两种模式**（示例）  
   - 默认：跑 HTTP API  
   - `--migrate`（或等价环境变量）：只跑 DbUp 后退出码 0/非 0  

3. **编排顺序**（Docker Compose / K8s）  
   - 数据库 healthy → **migrate Job 成功结束** → 再启动 / 扩容 API  
   - Compose：`depends_on: migrate: condition: service_completed_successfully`（或等价）  

4. **本地调试**  
   - `docker compose run --rm <service>-migrate`  
   - 或 `dotnet run --project ... -- --migrate`  

## 后果

**正向**

- 多副本安全；发布门禁清晰（迁移不过不发 API）
- 镜像仍只有一份，运维简单

**代价**

- Compose / Chart 多一个 migrate 服务定义
- 开发需记住「改 SQL 先 migrate」

## 反模式（禁止）

- 在 `Program.cs` 主路径无条件 `EnsureDatabase` + 全量 migrate 后直接 `Run`
- 多个 API 副本同时对同一库做「启动即迁移」且无分布式锁设计

## 落地检查清单

- [ ] 迁移脚本为嵌入资源，版本化文件名（如 `001_init.sql`）
- [ ] CI/CD 或 compose 在 API 之前跑 migrate
- [ ] API 启动仅假设 schema 已就绪（可做轻量连通性检查，不做 schema 变更）
- [ ] 组织模板（GameTemplate）与新服务脚手架默认带 migrate Job

## 参考实现

- MP：`mp-auth-migrate` + `docker compose` 依赖顺序  
- Mail：`mail-migrate` 服务与 API 同 Dockerfile  

## 修订历史

| 日期 | 说明 |
|------|------|
| 2026-09-12 | 初版 Accepted（框架期两大问题之一） |
