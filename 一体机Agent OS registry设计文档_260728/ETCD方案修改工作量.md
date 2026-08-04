# rqlite → etcd 迁移工作量评估

> 评估注册中心把存储后端从 **rqlite（分布式 SQL）** 替换为 **etcd（分布式 KV）** 需改动的功能模块与代码量。基于对 `a2x_registry-0.3.3` 实际代码的静态分析。

## 1. 背景与关键结论

- rqlite 是**分布式 SQL 数据库**（SQLite 方言 + Raft 复制）；etcd 是**分布式键值存储**（KV + 租约 lease + watch）。二者不是同类替换。
- 注册中心当前存储抽象是 **“SQL 字符串”级**：`common/db.py` 暴露 `Backend.execute(sql) / query(sql)`，`A2X_REGISTRY_DB_KIND` 只在 `memory / sqlite / rqlite` 之间切换——**三者都是 SQL**。
- etcd 无 SQL，会**打破这个接口**。因此迁移**不是“加一个 etcd 后端”那么简单**，而是：
  1. 把存储抽象从 SQL 接口**上抬为仓储（repository）接口**（按领域对象 get/put/list/delete/txn/lease）；
  2. **重写所有 SQL 调用点**为 KV 操作；
  3. 把 SQL 的 `WHERE 过滤 / ORDER / COUNT / JOIN` 用**前缀扫描 + 应用层过滤**或**自维护二级索引键**重实现；
  4. 属主校验、租约改用 **etcd txn / lease**。

## 2. 耦合面（实测：发 SQL 的文件）

| 文件 | `.execute/.query` 调用点 | 文件行数 |
|---|---:|---:|
| `common/db.py`（存储抽象 + `init_schema` 建 4 张表） | 5（抽象自身定义） | 371 |
| `register/service.py` | 11 | 2008 |
| `image/service.py` | 9 | 341 |
| `image/router.py` | 1 | 133 |
| `instance/service.py` | 4 | 269 |
| `vector/*` | 3 | — |

`init_schema` 建表：`registry_meta` / `service` / `image` / `instance`（+ 索引）。

## 3. 功能模块工作量拆分

| 功能模块 | 修改内容 | 预计代码量 |
|---|---|---:|
| **存储抽象层**<br>`common/db.py` | 把 `execute(sql)/query(sql)` 的 SQL 连接替换为 etcd 客户端封装，并**上抬为仓储接口**（get/put/list/delete/txn/lease）；新增 etcd 后端 | 300–400（重写） |
| **数据模型 / 键布局**<br>（替代 `init_schema` 4 张表） | 设计 etcd 键前缀布局；为原 SQL `WHERE` 过滤维护**二级索引键**；替换 DDL | 50–100 |
| **注册核心**<br>`register/service.py`（2008 行） | 11 处 SQL→仓储调用；属主校验 / 状态 / 租约改 **etcd txn + lease**；过滤 / 排序改应用层 | 200–600 |
| **镜像**<br>`image/service.py` + `router.py` | 10 处 SQL→KV；镜像列表 / 分页 / 按框架过滤改**前缀扫描 + 索引** | 120–180 |
| **实例**<br>`instance/service.py` | 4 处 SQL→KV；实例存活改 **etcd lease**（更自然） | 60–100 |
| **向量检索**<br>`vector/*` | 3 处 SQL；一体机模式不用，**可跳过** | 0–40 |
| **启动 / 配置**<br>`backend/startup.py` + `__main__.py` | 去掉 `init_schema`；`A2X_REGISTRY_DB_*` 改 etcd endpoint / 证书；建连 | 50–80 |
| **部署**<br>agent-os `module.sh` + `build.sh` | 删 rqlite rpm 下载 / 安装、rqlited unit、single-node、等选主；改对接平台 etcd（endpoint / 证书 env） | ~40（删多于加） |
| **文档 / 契约**<br>OpenAPI + 设计文档 | DB 说明、部署方案、环境变量表更新 | 30–50（文档） |

## 4. 总计（向上限估计）

| 功能模块 | 上限代码量 |
|---|---:|
| 存储抽象层 | 400 |
| 数据模型 / 键布局 | 100 |
| 注册核心 | 600 |
| 镜像 | 180 |
| 实例 | 100 |
| 向量检索 | 40 |
| 启动 / 配置 | 80 |
| 部署 | 40 |
| 文档 / 契约 | 50 |
| **合计（上限）** | **约 1600 行** |

> 说明：上限已计入 `register/service.py` 查询/租约深度耦合、以及 vector 不跳过的情形。常态区间约 **780–1250 行**；此处按**上限约 1600 行**估。行数不含“架构上抬”带来的调用点签名连锁改动的隐性成本。

## 5. 风险与建议

- **规模**：中等偏大重构，风险集中在 `register/service.py`（2008 行）的查询与租约逻辑。
- **卡点不在行数，在架构**：核心是把“SQL 字符串抽象”上抬为“仓储抽象”，牵动全部调用点——这是架构级改动，不是局部替换。
- **正确性风险**：SQL 的过滤 / 排序 / 计数 / 事务在 etcd 需用应用层扫描 + 索引 + txn 重实现，易引入正确性缺陷，需完整回归持久层测试。
- **建议两步走**：
  1. 先把持久层从“SQL 字符串”解耦为 **repository 接口**（内部仍走 rqlite，行为不变、可回归）——即便不换 etcd 也有独立价值；
  2. 再加 etcd 实现，按 `DB_KIND=etcd` 灰度上线。
  将“架构解耦”与“换存储底座”拆成两个可验证阶段，避免一次性大改带来的集中风险。
