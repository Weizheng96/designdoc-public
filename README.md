# 一体机 Agent OS Registry 设计文档（公开）

一体机场景下九问 Agent 注册中心的设计与实现文档，公开发布用于与协作方讨论。

## 内容

[一体机Agent OS registry设计文档/](./一体机Agent%20OS%20registry设计文档/)

| 文件 | 内容 |
|------|------|
| [一体机Agent OS registry设计文档.md](./一体机Agent%20OS%20registry设计文档/一体机Agent%20OS%20registry设计文档.md) | 功能列表 · 整体视图 · 场景逻辑视图 · 注册表示例 |
| [一体机Agent OS registry实现文档.md](./一体机Agent%20OS%20registry设计文档/一体机Agent%20OS%20registry实现文档.md) | 项目架构 · 各模块功能与接口 · 数据库改造（SQLite / rqlite）|
| [registry_openapi.yaml](./一体机Agent%20OS%20registry设计文档/registry_openapi.yaml) | 全部 REST 接口的 OpenAPI 3.0 契约 |
| [方案对比.md](./一体机Agent%20OS%20registry设计文档/方案对比.md) | 镜像元数据归属两方案对比（元戎是否额外维护镜像注册表）|

## 关联

- 通用服务列表接口契约（`GET /api/datasets/{dataset}/services` 分页约定）见 [A2X-registry/docs/backend_api.md](https://github.com/Weizheng96/A2X-registry/blob/main/docs/backend_api.md)。
- 元戎函数 / 沙箱 API 见 [openyuanrong 官方文档](https://docs.openyuanrong.org/)。
