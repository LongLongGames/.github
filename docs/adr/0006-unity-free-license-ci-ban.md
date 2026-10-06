# ADR-0006：不在 CI 中使用 GameCI / 云端 Unity 构建

- **状态**：已采纳
- **日期**：2026-10-07
- **范围**：`game-match3-client` 及同组织其它 Unity 客户端仓库
- **相关**：LocalCDN 发版约定、`version.json`、GameLauncher 下载链路

## 背景

希望用 GitHub Actions 自动打玩家包与 AssetBundle（AB），常见方案是 GameCI
（`game-ci/unity-builder` 等）在托管 runner 上跑 Unity Editor。

客户端使用的是 **Unity Personal（免费许可）**。

## 决策

**不在 GitHub Actions（或任何云 CI）中运行 Unity Editor 构建。**

不引入 GameCI 作为正式发版依赖。

Unity 相关产物仅通过：

1. **编辑器 `MenuItem`**（主路径），或
2. **本机已激活许可下的 `-batchmode -executeMethod`**（可选本地脚本）

完成构建；随后用本机/非 Unity CI 脚本生成 `version.json` 并同步到 LocalCDN
（`download/`、`ab/`、`config/`）。

## 原因

1. **许可与激活**  
   GameCI 在 CI 中需要有效的 Unity 许可（Personal 通常依赖 `.ulf` / 账号类 secrets）。
   Personal 面向本机激活，在无图形界面、多机、共享 runner 上激活与续期不稳定，
   运维成本高，不适合作为组织标准发版链路。

2. **与「免费可复现发版」目标冲突**  
   云端 Unity 构建还受 GitHub Actions 私有库分钟数、制品体积、构建时长等约束。
   在 Personal 已限制云端使用的前提下，再维护一套脆弱的 CI 许可配置，收益不足。

3. **架构已具备解耦**  
   Launcher 只消费 CDN 上的静态清单与包路径，不关心包由谁构建。
   因此放弃云端 Unity CI 不影响下载与更新设计。

## 后果

### 正面

- 发版不依赖 Unity 云许可与 GameCI 激活流程
- 构建环境与开发者本机一致，问题更好复现
- CI 可专注非 Unity 步骤（校验、生成 version.json、上传 CDN）

### 负面

- 玩家包 / AB 不能在每次 push/PR 上全自动云端产出
- 发版依赖本机点菜单或跑本地脚本
- 需约定固定输出目录，便于脚本与 LocalCDN 路径对齐

### 明确不做

- 不在 `.github/workflows` 中配置需要 CI 内激活 Unity 的 GameCI job
- 不把「GitHub 托管 runner + Unity Personal」当作可行发版方案

## 可选演进（非当前范围）

若未来使用 Unity Plus/Pro 序列号或浮动许可，或自建已激活的 self-hosted runner，
可另开 ADR 评估是否恢复部分自动化构建。在此之前保持本决策。