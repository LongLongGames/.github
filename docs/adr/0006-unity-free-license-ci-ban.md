# ADR-0006：不在 CI 中使用 GameCI / 云端 Unity 构建

- **状态**：已采纳
- **日期**：2026-10-07
- **范围**：`game-match3-client` 及同组织其它 Unity 客户端仓库
- **相关**：LocalCDN 发版约定、`version.json`、GameLauncher 下载链路

## 背景

希望用 CI 自动打玩家包与 AssetBundle（AB）。常见方案包括：

- GitHub Actions + GameCI（`game-ci/unity-builder`）
- 自建 / 云托管 GitLab CI + GameCI Docker 镜像
- 其他云 CI 跑 Unity Editor

客户端使用的是 **Unity Personal（免费许可）**。

## 决策

**不在任何 CI（GitHub Actions、GitLab CI 云端/自建 Docker 方式、其他云 CI）中运行需要重新激活 Unity 许可的构建。**

不引入 GameCI（或任何依赖 CI 内激活 Personal 许可的方案）作为正式发版依赖。

Unity 相关产物仅通过：

1. **编辑器 `MenuItem`**（主路径），或
2. **本机（或固定打包机）已激活许可下的 `-batchmode -executeMethod`**（可选本地/固定机器脚本）

完成构建；随后用本机/非 Unity CI 脚本生成 `version.json` 并同步到 LocalCDN（`download/`、`ab/`、`config/`）。

## 原因

### 1. 许可与激活（核心卡点）

Unity Personal 面向**本机一次性激活**（Named User + 登录 Unity ID），官方已不再支持可靠的手动/离线激活。

无论 CI 是：

- GitHub Actions 托管 runner
- GitLab.com 托管 runner
- **自建 GitLab + Docker / GameCI 镜像**

只要每次构建都要在新容器/新机器指纹上重新激活，就会遇到：

- 激活失败、续期失败、机器绑定冲突
- 需要长期维护 `UNITY_EMAIL` / `UNITY_PASSWORD` / `.ulf` 或提取的 serial 等 secrets
- 2FA、账号安全策略、Unity 服务端变动随时导致流水线挂掉
- 并发构建更容易撞席位限制

这些成本远高于「本机点一下菜单」或「固定机器跑脚本」。

**结论：GitLab 并不比 GitHub Actions 更友好。**  
只要还是「CI 里重新激活 Personal」，无论平台是 GitHub 还是 GitLab（云或自建 Docker），都不适合作为组织标准发版链路。请彻底死心。

### 2. 与「免费可复现发版」目标冲突

云端 Unity 构建还受分钟数、制品体积、构建时长、网络稳定性等约束。  
在 Personal 已限制稳定云端使用的前提下，再维护一套脆弱的 CI 许可配置，收益不足。

### 3. 架构已具备解耦

Launcher 只消费 CDN 上的静态清单与包路径，不关心包由谁构建。  
因此放弃云端/CI 内 Unity 构建，不影响下载与更新设计。

### 4. 与传统公司实践一致

传统公司做法是：一台固定 Windows 打包机 → 安装并激活 Unity → Jenkins 只负责触发已激活的 Editor。  
这与本 ADR 的「MenuItem / 本机 batchmode」路径本质相同，**没有额外 license 问题**。  
试图把这台固定机器换成「每次重新激活的 CI 容器」才是引入问题的根源。

## 后果

### 正面

- 发版不依赖 Unity 云许可与 GameCI 激活流程
- 构建环境与开发者本机（或固定打包机）一致，问题更好复现
- CI 可专注非 Unity 步骤（校验、生成 version.json、上传 CDN）
- 免费 Personal 下最干净、最稳的方案

### 负面

- 玩家包 / AB 不能在每次 push/PR 上全自动云端产出
- 发版依赖本机点菜单或跑本地/固定机器脚本
- 需约定固定输出目录，便于脚本与 LocalCDN 路径对齐

### 明确不做

- 不在 `.github/workflows` 中配置需要 CI 内激活 Unity 的 GameCI job
- 不在 GitLab CI（云端或自建 Docker 方式）中配置需要重新激活 Personal 许可的 Unity 构建
- 不把「任何托管 runner / 临时容器 + Unity Personal」当作可行发版方案
- 不投入时间去「折腾」Personal 在 CI 里的激活稳定性

## 可选演进（非当前范围）

仅在以下情况可另开 ADR 评估是否恢复部分自动化构建：

1. 升级到 Unity Plus/Pro 序列号或浮动许可（License Server）
2. 自建**固定**、已提前激活好的 self-hosted runner（本质仍是「传统打包机」模式，而非每次重新激活）

在此之前保持本决策。请读者对「免费 Personal + CI 自动打 exe/AB」彻底死心。