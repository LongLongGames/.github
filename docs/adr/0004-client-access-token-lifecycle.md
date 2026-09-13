# ADR-0004：客户端 Access Token 生命周期与鉴权失败处理

- **状态：** Proposed  
- **日期：** 2026-09-14  
- **影响面：** 所有 Unity 客户端（HybridCLR / VContainer 热更）、依赖 MP JWT 的 Game.* 联调与正式包  
- **相关：** [ADR-0002](./0002-aot-json-and-jwt.md)（服务端签发与验签）、MP `/api/v1/auth/login`、各游戏 `IAuthService` / HTTP 封装  

## 背景

平台约定：**MP 签发 HS256 JWT**，游戏服与中台共用 Secret 验签（见 ADR-0002）。  
客户端登录成功后将 `access_token` 持久化，后续请求带 `Authorization: Bearer <token>`。

match3-client 联调中暴露的典型错误模型：

1. **「本地有字符串 = 已登录」**  
   启动仅 `PlayerPrefs` 读出非空 token 即进入 Home，**不向服务端探活**。
2. **Token 失效后仍当会话有效**  
   过期、服务端换 `JWT_SECRET`、清库、踢下线等之后，客户端继续进主界面。
3. **401 被业务层吞掉**  
   `profile` / `state` / `level/clear` 等返回 401 时，仅打日志并回退本地空进度 → 玩家看到「又变第一关」，服务端日志出现连环 401。
4. **无统一登出路径**  
   失效 Token 不会被清除，下次启动重复 1–3。

上述问题在 Editor 重开、服务重启、跨日会话中极易复现，且会污染网关/游戏服访问日志。

## 决策

### 1. 登录态定义

- **唯一可信登录态** = 内存中持有的 Access Token **且** 通过服务端认可（探活成功，或业务请求未因鉴权失败）。  
- **禁止**将「持久化介质中存在非空 token 字符串」单独等同于 `IsLoggedIn` 并进入需鉴权的主流程。

### 2. 持久化

- Key 约定：`access_token`（或各项目统一常量，但全公司语义一致）。  
- 介质：当前允许 `PlayerPrefs`；写入与删除后必须 **`Save()`**（或等价刷盘）。  
- 登录成功：内存赋值 + 持久化。  
- 登出 / 判定失效：内存清空 + 删除持久化 key。  
- **禁止**调试或业务代码绕过 `IAuthService`（或等价门面）直接写死假 token 并当作已登录（调试须走登录或显式 mock 且不得进正式包）。

### 3. 启动顺序（冷启动 / 热更入口之后）

在版本检查与必要配置加载之后：

```
TryRestoreToken()          // 仅恢复到内存，不判定已登录
若内存无 Token → 进入 Login
若内存有 Token → ValidateSession（轻量鉴权请求）
    成功 → 进入 Home（或项目主界面）
    401 / 明确未授权 → Logout → Login
    其它网络/5xx → 不得当作已登录进入主界面（默认回 Login 或可重试页；禁止用空本地进度冒充服务器进度）
```

- **ValidateSession** 推荐调用与主界面相同的轻量鉴权接口（如 `GET .../user/profile`），避免额外专用接口也可。  
- 探活 **不是** 刷新 Token；在未引入 Refresh Token 前，探活只回答「当前 access 是否仍被服务端接受」。  
- 每人每次冷启动多一次成功探活的成本可接受；相对「假登录连环 401」更优。

### 4. 运行中 HTTP 鉴权失败（强制）

- 所有 **携带 Bearer** 的请求若响应 **401**：  
  1. 触发统一 `Unauthorized` 回调 / 事件（HTTP 层）  
  2. 抛出可识别的未授权异常（便于上层区分网络错误）  
  3. **Logout**（清内存 + 持久化）  
  4. 导航至 **Login**（需防抖，避免并发 401 多次跳转）  
- 业务层（Player / Leaderboard 等）**不得**在 401 时静默回退本地空数据并继续主流程。  
- 403（权限不足但会话仍有效）与 401 区分处理；本 ADR 约束的是 **401 会话失效**。

### 5. 与服务端的契约

- 客户端不解析 JWT 业务权限；过期可作本地提示优化，**不能替代**服务端 401。  
- 服务端须对无效/过期 Token 稳定返回 **401**（错误体遵循 ADR-0002，禁止匿名类型）。  
- 未来若引入 Refresh Token：Access 短有效期 + Refresh 旋转；本 ADR 的「401 → Logout」仍适用（Refresh 失败再登出）。Refresh 细节另立 ADR。

### 6. 客户端模块边界（Unity）

- `IAuthService`：`LoginAsync` / `TryRestoreToken` / `ValidateSessionAsync` / `Logout` / `IsLoggedIn`。  
- `IHttpClient`（或等价）：持有内存 Token、附加 Header、401 时事件 + 专用异常。  
- `IVersionService`（或独立 `ITokenStore`）：只负责持久化读写，不解释登录态。  
- 流程控制器（如 `AppFlow`）负责：启动探活分支、订阅 401、跳转 Login/Home。

## 后果

**正向**

- 消除「假登录 → 空进度 / 假第一关 → 服务器 401 刷屏」  
- 登录态与 MP/游戏服验签语义一致，跨游戏可复用同一套客户端约定  
- 调试与正式路径分离清晰（必须经 Auth 门面）

**代价**

- 冷启动多一次鉴权 RTT  
- HTTP 与 Flow 需接线（事件/异常），不能只在单个 API 调用处打补丁  
- 弱网下「有旧 Token 但探活失败」会回登录，需产品文案（网络异常 vs 登录过期）可选优化

## 反模式（禁止）

- `IsLoggedIn => !string.IsNullOrEmpty(PlayerPrefs.GetString("access_token"))` 且无服务端探活即进 Home  
- 401 后继续使用原 Token 重试业务写接口  
- 401 时仅用本地缓存进度填充 UI，不登出  
- 调试直接 `PlayerPrefs.SetString("access_token", "test_token")` 当作已登录（且进入需鉴权流程）  
- 每个业务 Service 各自实现一套「失败就当游客」而不走统一 Logout  

## 落地检查清单

- [ ] 登录成功：内存 + 持久化；登出：双清并 `Save`  
- [ ] 启动：`Restore` → 有 Token 则 `ValidateSession` → 再 Home/Login  
- [ ] HTTP：鉴权请求 401 → 事件 + 未授权异常 → Logout → Login（防抖）  
- [ ] 业务层不吞 401 为「本地空进度」  
- [ ] Editor/开发包调试经 `IAuthService` 或显式 Debug 门面，不写死假 JWT 进主流程  
- [ ] 与 MP 联调：过期 Token、改 Secret、无 Header 均应 401 且客户端回登录  
- [ ] 服务端错误体符合 ADR-0002  

## 参考实现

- match3-client（修复后）：  
  - `Auth/AuthService.cs`（`ValidateSessionAsync` / `Logout`）  
  - `Network/HttpClientService.cs`（401 → `Unauthorized` + `UnauthorizedException`）  
  - `AppFlow/AppFlowController.cs`（启动探活 + 401 强制回 Login）  
  - `Services/VersionService.cs`（`access_token` + `PlayerPrefs.Save`）  
- 服务端：ADR-0002、MP `SimpleJwt`、Game 网关 Bearer 验签  

## 修订历史

| 日期 | 说明 |
|------|------|
| 2026-09-14 | 初版 Proposed（match3 联调：假登录与连环 401 复盘） |
```
