# ADR-0002：Native AOT 下的 JSON 错误体与 JWT 验签

- **状态：** Accepted  
- **日期：** 2026-09-12  
- **影响面：** 所有 `PublishAot=true` + Minimal API 的服务与共用 JWT 的游戏组件  
- **相关：** MP `SimpleJwt`、Mail、match3 等 Game.* 服务

## 背景

框架统一 **.NET Native AOT** 后，暴露两类高影响问题（曾在 Mail 联调中集中爆发，其它仓库 happy path 下可长期潜伏）：

### 问题 A：匿名类型错误响应

`System.Text.Json` 源生成（`JsonSerializerContext`）**只序列化已登记类型**。

```csharp
// AOT 下错误分支一旦执行 → NotSupportedException → 本该 400/401 变成 500
return Results.BadRequest(new { error = "game_id required" });
```

成功路径类型登记完整时，联调「看起来全绿」；异常入参第一次命中即炸。

### 问题 B：JWT 验签实现分叉

平台约定为 MP 签发、**共享 Secret 的 HS256**。  
MP 使用手写 **`SimpleJwt`**（无 kid、无 IdentityModel）。

若业务服务改为 `JwtSecurityTokenHandler` + `Microsoft.IdentityModel.*` 手动 `ValidateToken`，在精简 host（`CreateSlimBuilder`）与无 `kid` 的 Token 上，易出现如 `IDX10517`（签名校验失败 / kid missing）等与平台 Token 不兼容的行为。

这 **不是**「AOT 与 JWT 天生冲突」：官方 `AddJwtBearer` 中间件路径有 AOT 支持工作；冲突来自 **未与平台 SimpleJwt 对齐的手写 IdentityModel 路径**。

## 决策

### 1. JSON / 错误体

- 禁止在 API 返回值中使用 **匿名类型** 与未登记的 `object` 图  
- 统一例如 `ErrorResponse { string Error }`（或等价），并 `[JsonSerializable]` 登记进各服务 `AppJsonContext`  
- 推荐显式：

```csharp
Results.Json(new ErrorResponse { Error = "unauthorized" },
    AppJsonContext.Default.ErrorResponse,
    statusCode: 401);
```

### 2. JWT

- **签发：** 仅 MP（或未来统一 Auth）使用与 `SimpleJwt` 兼容的 HS256  
- **校验：** 游戏与组件服务使用与 MP **同构**的手写 HS256 验签（可复制 `SimpleJwt` 逻辑），共用 `JWT_SECRET` / `Auth__JwtSecret`  
- **不要**在业务服务中引入 `System.IdentityModel.Tokens.Jwt` 仅做手动 `ValidateToken`，除非团队统一评估并回归 AOT + 与 MP Token 兼容性  
- 本地 Sample 可自签 HS256，生产以 MP Token 为准  

### 3. 测试

- 每个服务至少具备 **happy + bad path** 的 HTTP smoke（401/400/404 必须返回且 body 为合法 JSON）  
- 推荐：相关路径 push/PR 跑 smoke；`v*` tag 再跑（发布门禁）  
- 报告：CI Artifact（JSON/JUnit）与 Commit 绑定  

## 后果

**正向**

- 错误路径在 AOT 下行为可预测  
- 全公司 Token 互通，减少验签库版本坑  
- bad path 测试能提前引爆「匿名类型」类问题  

**代价**

- 错误 DTO 需维护并登记 Context  
- 新同事需阅读本 ADR，不能随手 `new { error = ... }`

## 反模式（禁止）

- `Results.BadRequest(new { error = "..." })` 等匿名对象  
- 业务服务依赖 IdentityModel 手写校验、与 MP Token 头/claim 不一致  
- 只测 happy path 就上生产 AOT 镜像  

## 落地检查清单

- [ ] 全局搜 `new { error` / `Results.BadRequest(new {` 并替换  
- [ ] `AppJsonContext` 含所有响应类型（含错误）  
- [ ] 验签与 MP `SimpleJwt` 对齐（claim 至少支持 `sub`）  
- [ ] smoke：无/错密钥、错 JWT、缺字段、不存在资源  
- [ ] CI 上传 `test-results/`  

## 参考实现

- MP：`src/MP.Auth/Infrastructure/Jwt/SimpleJwt.cs`  
- Mail：`Auth/SimpleJwt.cs`、`ErrorResponse`、`server/scripts/smoke_test.py`、`.github/workflows/api-smoke.yml`  
- Mail 文档：`docs/api-testing.md`  

## 修订历史

| 日期 | 说明 |
|------|------|
| 2026-09-12 | 初版 Accepted（框架期两大问题之一） |
