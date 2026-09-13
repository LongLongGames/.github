# ADR-0003：Monorepo 中 OpenUPM 包与 GHCR/Server 发布的 Git Tag 隔离

- **状态：** Accepted  
- **日期：** 2026-09-14  
- **影响面：** 所有在同一 Git 仓库中同时发布 Unity UPM 包（OpenUPM）与 Server/Docker 镜像（GHCR / Actions）的 monorepo 组件（如 BugReport、后续同类组件）  
- **相关：** OpenUPM `data/packages/*.yml`、`package.json`、`release-server.yml`、GHCR 推送工作流

## 背景

公司部分组件采用 monorepo 结构：同一仓库既包含 Unity 客户端包（通过 OpenUPM 分发），也包含服务端代码（通过 GitHub Actions 构建 Docker 镜像并推送到 GHCR）。

早期实践中，Server 侧使用了类似 `server-v1.0.0` 或无前缀的 `v1.0.0` 风格 tag。OpenUPM 的构建流水线默认会扫描仓库中的所有 Git tag，并尝试将其解释为包版本。结果：

- Server 的 tag（如 `server-v1.0.0`）被 OpenUPM 误识别为包版本，导致构建失败、页面污染或元数据异常。
- 包在 OpenUPM 官网的页面可能暂时消失或无法正常编辑（常因 PR 临时分支被删除后 Edit 链接指向 404）。
- 职责混淆：有人误把 `gitTagPrefix` 写入 Unity 的 `package.json`，但这并不是 Unity 官方字段。

OpenUPM 官方明确支持 monorepo 场景，并通过包元数据 YAML 中的 `gitTagPrefix` 字段做 **字面前缀过滤**（不是正则），只处理属于该包的 tag 家族。

## 决策

### 1. 严格的 Tag 命名空间隔离

| 用途 | Tag 格式示例 | 消费方 |
|------|--------------|--------|
| Unity / OpenUPM 包 | `com.setsuodu.bugreport/1.0.0` | OpenUPM 构建流水线 |
| Server / Docker | `server/v1.0.0` | GitHub Actions → GHCR |

- 包 tag **必须** 以包名 + `/` 为前缀（与 OpenUPM 推荐的 monorepo 实践一致）。
- Server tag **必须** 使用独立前缀（推荐 `server/`），且 **绝不** 与包 tag 前缀重叠。

### 2. `gitTagPrefix` 只属于 OpenUPM 元数据

- **禁止** 在 Unity `package.json` 中添加 `gitTagPrefix` 字段。  
  `package.json` 只保留官方字段，例如：

  ```json
  {
    "name": "com.setsuodu.bugreport",
    "version": "1.0.0"
  }
  ```

- `gitTagPrefix` 写在 OpenUPM 的包元数据文件中（`openupm/openupm` 仓库的 `data/packages/<package-name>.yml`）：

  ```yaml
  name: com.setsuodu.bugreport
  repoUrl: 'https://github.com/...'
  gitTagPrefix: 'com.setsuodu.bugreport/'
  # 可选：gitTagIgnore、minVersion 等按需配置
  ```

### 3. 已有包的修改流程（不要重新 /packages/Add）

1. 进入 [openupm/openupm](https://github.com/openupm/openupm) 的 **默认分支（master）**。
2. 确认 `data/packages/<package-name>.yml` 是否存在。
   - **存在**：直接 Edit 该文件 → 修改 `gitTagPrefix` → 新建分支 → 开 PR。
   - **不存在**（且官网页面也消失）：才考虑重新走 `/packages/Add`。
3. 注意：旧 PR 页面上的 “Edit file” 常指向已删除的临时分支（如 `patch-3`），会 404。必须从默认分支重新编辑。
4. PR 合并后，OpenUPM 会重新触发对符合前缀的 tag 的构建监控。

### 4. 发布顺序建议

- 先在本仓库打符合前缀的包 tag（如 `com.setsuodu.bugreport/1.0.0`），并确保 `package.json` 的 `version` 与 tag 中的版本号一致。
- 再打 Server tag（`server/v1.0.0`）触发 Actions / GHCR。
- 修改 OpenUPM YAML 的 PR 合并后，流水线会自动拾取已存在的、符合 `gitTagPrefix` 的 tag。

## 后果

**正向**

- OpenUPM 与 GHCR 发布完全解耦，互不污染。
- 符合 OpenUPM 官方 monorepo 最佳实践，后续包可直接复用同一模式。
- `package.json` 保持干净，只表达 Unity 包元数据。

**代价**

- 需要维护两套 tag 命名约定，并在 CI 文档 / README 中明确写出。
- 历史被污染的 tag 可能需要人工清理或通过 `gitTagIgnore` / `minVersion` 过滤。
- 修改已有包必须走 OpenUPM 的 YAML PR 流程，而不是重新 Add。

## 反模式（禁止）

- 在 `package.json` 中写入 `gitTagPrefix`。
- 使用无前缀或与包名前缀冲突的 tag（如裸 `1.0.0`、`v1.0.0`、`server-v1.0.0` 被 OpenUPM 误扫）。
- 发现 404 后直接重新 `/packages/Add`，而不先检查默认分支上 YAML 是否仍在。
- 让 Server 的 Actions 工作流去修改或依赖 OpenUPM 的 tag 规则。

## 落地检查清单

- [ ] 本仓库 Git tag 已按「包前缀 / Server 前缀」重新划分（历史污染 tag 已处理或忽略）
- [ ] `package.json` 无任何 OpenUPM 私有字段
- [ ] OpenUPM `data/packages/<name>.yml` 中已设置正确的 `gitTagPrefix`
- [ ] Server 相关 workflow（如 `release-server.yml`）只响应 `server/*` 类 tag
- [ ] 组件 README 或发布文档明确写出两套 tag 约定
- [ ] 新 monorepo 组件脚手架默认带上上述约定

## 参考

- OpenUPM 官方文档：Adding UPM Package / monorepo 与 `gitTagPrefix` 说明  
- OpenUPM 仓库：`data/packages/*.yml` 元数据位置与修改流程  
- 本决策来源：BugReport 组件 OpenUPM 与 Server tag 冲突的排查与纠正过程

## 修订历史

| 日期 | 说明 |
|------|------|
| 2026-09-14 | 初版 Accepted（总结 monorepo 下 OpenUPM + GHCR 双发布 tag 隔离方案） |
