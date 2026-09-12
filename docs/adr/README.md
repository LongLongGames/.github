# Architecture Decision Records（LongLongGames）

影响 **几乎全部框架衍生项目** 的工程决策在此归档。各游戏/组件仓库实现时必须遵守；细则可链到本目录。

| 编号 | 标题 | 状态 |
|------|------|------|
| [ADR-0001](./0001-dbup-migrate-job.md) | 数据库迁移与 API 进程分离（DbUp） | Accepted |
| [ADR-0002](./0002-aot-json-and-jwt.md) | Native AOT 下的 JSON 错误体与 JWT 验签 | Accepted |

## 如何新增 ADR

1. 复制编号递增，文件名 `NNNN-slug.md`
2. 状态：Proposed → Accepted / Deprecated
3. 在本 README 表格登记
4. 在组织 profile README「工程规范」中加一行链接

## 提交位置（组织约定）

见仓库根目录 [COMMIT.md](../COMMIT.md)（打包说明里是 `COMMIT.md`）。
