# ETCD 兼容分析

> 目标：rqlite 与 etcd 双支持（依赖倒置抽象），etcd 复用元戎在一体机提供的 etcd、仅一体机可选。**核心原则：etcd 实现的接口，其功能效果与 SQL（rqlite）版一致**——由双后端契约测试守门。一体机只需行级 CRUD + 列表，无 JOIN / 复杂查询。基于 `feature/Agentregistry-dev` 当前实现。决策逐条见 [§4](#4-决策记录)。

## 1. 当前 SQL 调用接口 · 一体机涉及哪些

所有 SQL 最终经 `common/db.py` 的 `Backend.execute(sql)` / `query(sql)`；上层 `register/service.py::RegistryTableService` 把它们封成方法。下表列这些方法（= SQL 接口），标注一体机（image / instance）是否用：

| 方法 | 作用 | SQL | 一体机 |
|---|---|---|:--:|
| `create_registry(name, kind)` | 建命名注册表（元数据） | `INSERT registry_meta` | ✅ 建 images / instances |
| `get_kind(name)` | 查注册表 kind | `SELECT registry_meta` | ✅ 内部 |
| `register(name, entry)` | upsert 一行 | `INSERT … ON CONFLICT` | ✅ |
| `get(name, sid)` | 取一行 | `SELECT * WHERE pk` | ✅ 内部（patch/deregister 校验） |
| `patch(name, sid, fields)` | 改字段 | `UPDATE … WHERE pk` | ✅ |
| `deregister(name, sid)` | 删一行 | `DELETE WHERE pk` | ✅ |
| `query(name, filter)` | 按等值 filter 列 | `SELECT * WHERE filter` | ✅ |
| `query_paginated(…)` | 过滤 + 排序 + 切片 + 计数 | `SELECT … WHERE/ORDER/LIMIT` + `COUNT` | ✅ **列表 / 筛选的唯一入口** |
| 数据集 / 服务 / 分类树 / 向量 / 租约（`list_services` / `get_taxonomy_state` / `list_datasets_with_counts` …） | A2X 通用模式 | 各类 SQL | ❌ 非一体机 |

**→ 一体机用 8 个方法**：`create_registry` / `get_kind` / `register` / `get` / `patch` / `deregister` / `query` / `query_paginated`。均为**按主键 `(registry, service_id)` 的行级 CRUD + 前缀列举 + 简单等值过滤 / 排序 / 分页**。

> 说明：`query_paginated` 是 image / instance **列表接口的唯一入口**（不带 `size` 时以 `limit=-1` 调它），筛选（`?framework/?user`）也走它——所以 **etcd 必须实现它**（不能不做）；只是一体机数据小，etcd 侧全量扫描后在应用层做过滤 / 排序 / 切片即可。

## 2. 一体机接口抽象成哪些接口

把上面 8 个方法收敛为一个 **`TableRepo` 抽象接口**（依赖倒置的约定）。**直接沿用现有方法名**，使 `RegistryTableService`（rqlite）天然满足；新增 `EtcdTableRepo` 实现同一接口。image / instance 只依赖 `TableRepo`：

| 抽象方法（= 现有方法名） | 语义 |
|---|---|
| `create_registry(name, kind)` | 声明命名注册表 |
| `get_kind(name)` | 查注册表 kind（不存在返 None） |
| `register(name, entry)` | **全量 upsert**（整行写入 / 覆盖；`entry` 含 `service_id`） |
| `get(name, service_id)` | 取一行 / None |
| `patch(name, service_id, fields)` | 改部分字段 |
| `deregister(name, service_id)` | 删一行 |
| `query(name, filter=None)` | 按等值 filter 列（`filter=None` 返全部） |
| `query_paginated(…)` | 过滤 + 排序 + 切片 + 计数（**两后端都实现**） |

**etcd 全部实现这 8 个方法，行为与 SQL 一致**（不再有 `raise NotImplementedError`）。**组合操作**（`set_default`、`register_instance` 的读旧合并 `created_at` 等）一律放 **service 层**、只用这 8 个原语拼 → 天然跨后端一致，etcd 侧零额外工作。

## 3. 工作量评估

### 3.0 整体架构

`a2x_registry/` 内：每个文件夹 = 模块，每个 `.py` = 子模块。存储相关的模块 / 子模块（**[新增]** 为本次要写的）：

```
a2x_registry/
├── common/                    模块：基础设施
│   └── db.py                  子模块：Backend.execute/query + connect() 工厂（rqlite/sqlite/memory）
├── register/                  模块：注册管理 + 表服务（存储抽象所在）
│   ├── service.py             子模块：RegistryTableService —— 现有 SQL 表服务（§1 的 8 方法）
│   ├── table_repo.py    [新增] 子模块：抽象接口 TableRepo（依赖倒置的约定）
│   ├── etcd_client.py   [新增] 子模块：etcd KV 连接封装（urllib）
│   └── etcd_repo.py     [新增] 子模块：EtcdTableRepo（实现 TableRepo）
├── image/  └ service.py       子模块：镜像服务，依赖 _table_svc
├── instance/ └ service.py     子模块：实例服务，依赖 _table_svc
└── backend/                   模块：装配与启动
    ├── __main__.py            子模块：读 env（DB_KIND / DB_ENDPOINT / ETCD_NAMESPACE / ETCD_TLS_*）
    └── startup.py             子模块：工厂选 repo 注入 image/instance service
```

依赖方向：`image/instance` → `TableRepo`（抽象）← `RegistryTableService`(rqlite) / `EtcdTableRepo`(etcd)；`backend` 工厂按 `A2X_REGISTRY_DB_KIND` 选注入。键布局：`/{namespace}/{registry}/{service_id}` → 行 JSON。两步走：**3.1 先抽接口 + rqlite 归位**（零行为改、可回归）→ **3.2 加 etcd**（按 `DB_KIND` 灰度）。

> **元戎 etcd 已核实**：etcd **3.5.24**、由元戎自带并启动、grpc-gateway 开着（元戎自带的 Python 冒烟测试就用 HTTP `/v3/kv/put|range|deleterange|txn`，key/value base64）。端点默认 `https://<ip>:32379`（单节点），compaction / 配额由元戎管。**接入决策**：客户端用 **urllib 打 `/v3` gateway**（与 rqlite 同为 stdlib、零新依赖）；连接安全 **配了证书走 mTLS、没配不认证**；namespace **按 key 前缀约定**（客户端给 key 加 `{namespace}/`）。完整决策见 [§4](#4-决策记录)。

### 3.1 重构抽象接口（rqlite 行为不变）

#### 3.1.1 `register/table_repo.py` **[新增]** — 抽象接口
所有上下层依赖的约定，先写。定义 `TableRepo` Protocol（沿用现有方法名，使 RegistryTableService 天然满足）：

```python
class TableRepo(Protocol):
    def create_registry(self, name: str, kind: str) -> None: ...
    def get_kind(self, name: str) -> Optional[str]: ...
    def register(self, name: str, entry: dict) -> dict: ...
    def get(self, name: str, service_id: str) -> Optional[dict]: ...
    def patch(self, name: str, service_id: str, fields: dict) -> dict: ...
    def deregister(self, name: str, service_id: str) -> bool: ...
    def query(self, name: str, filter: Optional[dict] = None) -> List[dict]: ...
    def query_paginated(
        self, name: str,
        filter: Optional[dict] = None,
        order_by: Sequence[Tuple[str, str]] = (),   # [(字段, "asc"|"desc"), …]
        limit: int = -1, offset: int = 0,
    ) -> Tuple[List[dict], int]: ...                 # (行列表, 过滤后总数)
```

各方法功能：
- `create_registry(name, kind)`：声明一个命名注册表（如 `images` / `instances`）及其 kind，启动时建；已存在则幂等。
- `get_kind(name)`：返回命名注册表的 kind，不存在返 None——供其余方法定位物理表 / 键前缀。
- `register(name, entry)`：**全量 upsert**（整行写入 / 覆盖，`entry` 含 `service_id`）；**不做 `created_at` 保留**——该逻辑由 service 层负责（见下"配套准则"）；返回存后的行。
- `get(name, service_id)`：按主键 `(name, service_id)` 取一行，不存在返 None。
- `patch(name, service_id, fields)`：**部分更新** `fields` 指定的列；条目不存在报错；返回更新后的行。
- `deregister(name, service_id)`：删一行，返回是否删除（幂等：本不存在返 False）。
- `query(name, filter)`：按**等值 filter** 返回匹配行列表；`filter=None` 返回该注册表全部行。
- `query_paginated(name, filter, order_by, limit, offset)`：过滤 + **结构化排序** + 切片，返回 `(行列表, 过滤后总数)`。`order_by` 是 `[(字段, "asc"|"desc"), …]`；字段为行 dict 的键，**JSON 内字段（如 `data.created_at`）** 由各后端各自处理（rqlite → `json_extract`，etcd → 从摊平的行 dict 取）。

**配套准则**：组合操作留 service 层——`register_instance` / `register_image` 的"读旧行 → 保留 `created_at` → 合并 → `register`"、`set_default` 的"清旧默认 + 置新默认"等，只用上述 8 原语拼；两后端共用同一段 service 代码 → 行为天然一致（`set_default` 现状即非原子的 `query`+多次 `patch`，etcd 照旧，不加事务）。

行数 **~40–60**。

#### 3.1.2 `register/service.py` — rqlite 实现归位
`RegistryTableService` 已实现这 8 方法。声明其满足 `TableRepo`（结构化类型 / 显式继承），补类型标注；`query_paginated` 去掉 `extra_where`/`extra_args` 两参、`order_by` 由裸 SQL 串改为结构化（内部据它拼 `ORDER BY`）。行数 **~15–30**。

#### 3.1.3 `image/service.py` · `instance/service.py` — 上层依赖抽象 + 去裸 SQL
- `_table_svc` 类型标注改为 `TableRepo`（依赖抽象）。
- `_IMAGE_ORDER` / `_INSTANCE_ORDER` 由 SQL 字符串改为**结构化排序规格** `[(字段,方向)]`。
- instance 的 `include_unhealthy=False` 由 `node NOT IN(失活集)` 的 `extra_where` 改为 `filter={'status':'运行'}`（随变更 2/3，心跳/失活集消失）。
行数 **~15–30**。

#### 3.1.4 `backend/startup.py` — 装配工厂
把"建 rqlite Backend → 造 RegistryTableService → 注入 service"抽成 `build_table_repo(cfg) -> TableRepo` 工厂函数（现状即此逻辑，仅归拢）。行数 **~10–20**。

> 小计 **~80–140 行**，行为不变（跑现有测试回归）。

### 3.2 对接 ETCD

#### 3.2.1 `register/etcd_client.py` **[新增]** — KV 连接封装
用 **`urllib`**（stdlib，与 rqlite `RqliteConnection` 一致、零新依赖）打 **etcd v3 HTTP/JSON gateway** `/v3/kv/*`，key/value 走 base64（与元戎自带冒烟测试同款）。**TLS 配置驱动**：配了客户端证书走 mTLS、没配不认证（`ssl.SSLContext`：`load_cert_chain(cert,key)` + `load_verify_locations(ca)`）；所有 key 统一加 `namespace/` 前缀。

```python
class EtcdClient:
    def __init__(self, endpoint: str, namespace: str,
                 ca: str = "", cert: str = "", key: str = "", timeout: float = 5): ...
    def get(self, k: str) -> Optional[dict]: ...
    def create(self, k: str, v: dict) -> bool: ...
    def put(self, k: str, v: dict, mod_revision: Optional[int] = None) -> bool: ...
    def delete(self, k: str) -> bool: ...
    def range(self, prefix: str) -> List[dict]: ...
```

各方法功能：
- `__init__`：装配 `urllib` opener 与 `ssl.SSLContext`（配 `cert/key/ca` → mTLS，没配 → 明文 / 不验）；连 `endpoint`（默认 `https://<ip>:32379`）；所有 key 自动加 `namespace/` 前缀。
- `get(k)`：`POST /v3/kv/range`（单键），返回值 JSON 或 None。
- `create(k, v)`：`POST /v3/kv/txn`，`compare create_revision==0` → **仅不存在时写**（幂等），已存在返 False。
- `put(k, v, mod_revision)`：`POST /v3/kv/put`；给了 `mod_revision` 则用 txn 做 **CAS 乐观锁**。
- `delete(k)`：`POST /v3/kv/deleterange`（单键）。
- `range(prefix)`：`POST /v3/kv/range`（`range_end` = 前缀尾字节 +1），返回前缀下所有值 JSON——供按 registry 列举。

读默认走 etcd **linearizable**（强一致，对齐 rqlite）。行数 **~60–90**（含 base64 + urllib/ssl 装配）。

#### 3.2.2 `register/etcd_repo.py` **[新增]** — EtcdTableRepo
`class EtcdTableRepo(TableRepo)`，用 `EtcdClient` 实现 8 方法；键 `/{ns}/{registry}/{service_id}`，行存 JSON：

- `register` = `create`/`put`（全量整行写）、`get` = `get`、`patch` = `get`+合并字段+`put`（带 `mod_revision` 乐观锁）、`deregister` = `delete`。
- `query(filter)` = `range(prefix)` 全量 → **应用层等值过滤**。
- `query_paginated(filter, order_by, limit, offset)` = `range(prefix)` 全量 → **应用层过滤 + 排序（按 order_by）+ 计数 + 切片（limit/offset）**。一体机数据小，全扫 + 内存处理无压力；行为与 rqlite 的 `WHERE/ORDER/LIMIT/COUNT` 一致。
- `create_registry` / `get_kind` = 元数据键 `/{ns}/_meta/{registry}`（`_` 前缀为内部保留，registry 名不以 `_` 开头，避免与 `/{ns}/{registry}/…` 撞）。
- **etcd 基础设施错误**（连不上 / 超时 / 配额）→ 映射为 `errors.py` 的外部依赖错误（HTTP `502`），与 rqlite 故障同档。

行数 **~180–240**。

#### 3.2.3 `backend/__main__.py` — 配置
`DB_KIND` 枚举加 `etcd`（**仅 appliance 模式**：`DB_KIND=etcd` 要求 `MODE=appliance`，否则报错）；复用 `A2X_REGISTRY_DB_ENDPOINT`（etcd 时必填，单 endpoint）；新增 `A2X_REGISTRY_ETCD_NAMESPACE`（key 前缀，**默认 `a2x-registry`**）+ `A2X_REGISTRY_ETCD_TLS_CA/CERT/KEY`（**三者配齐即对 etcd 走 mTLS，没配则不认证**）；校验。行数 **~20–30**。

#### 3.2.4 `backend/startup.py` — 工厂分支 + 启动失败处理
`build_table_repo`（3.1.4）加 `DB_KIND=etcd` 分支 → 造 `EtcdTableRepo`；etcd 时不建 rqlite 连接。**etcd 启动不可达即 fail-fast**（与 SQL 层一致，不带病启动）。行数 **~15–25**。

#### 3.2.5 `pyproject.toml` — 依赖
**无需改**——`urllib` 是 stdlib，不引任何新依赖（与 rqlite 后端一致）。行数 **0**。

#### 3.2.6 agent-os `deploy/agent-gateway/module.sh` · `build/build.sh` — 部署
etcd 模式：**不装 rqlite rpm、不起 rqlited**；注入 `DB_KIND=etcd` + `DB_ENDPOINT`（元戎 etcd）+ `ETCD_NAMESPACE` + 可选 `ETCD_TLS_CA/CERT/KEY`（连 etcd 的**客户端**证书，与注册中心自身服务端证书 `A2X_REGISTRY_TLS_*` 两套）；沿用现有 env 注入 + `_agentgw_has_systemd` 式开关。`up`/`install` 加一道 **ExecStartPre 探 etcd 连通性**（轮询 etcd `/health` 就绪再起 backend，避免启动竞态）。行数 **~30–50**。

#### 3.2.7 `tests/db/test_table_repo_etcd.py` **[新增]** — 契约测试
同一套契约测试**参数化跑 {SQL 后端, etcd}**（SQL 侧取 sqlite / rqlite 之一即可，二者共用 `RegistryTableService`），断言 8 方法两后端**行为一致**（含分页 / 排序 / 计数）；etcd 用 **CI Docker 真实实例**（`/v3` gateway，仿 `tests/start_rqlite_cluster.sh`），不用 mock；测试 seed 走 `register()`（后端无关，替代裸 SQL fixture）。**无 `NotImplementedError` 断言**（etcd 全实现）。行数 **~120–200**。

> 小计 **~305–435 行**。

**合计（3.1 + 3.2）≈ 385–575 行**：新增 4 个 `.py`（`table_repo` / `etcd_client` / `etcd_repo` / 测试），改动 5 个现有子模块（`register/service`、`image/service`、`instance/service`、`backend/__main__`、`backend/startup`）+ 2 个部署脚本；**零新增第三方依赖**（urllib）。etcd 硬限（单值 1.5MB / txn 128 op）对一体机小数据无碍。

## 4. 决策记录

**A 组（代码审查 → 已定）**

| # | 决定 |
|---|---|
| A#1 分页 | etcd **实现** `query_paginated`（应用层扫描 + 过滤 + 排序 + 切片）；**撤掉 `NotImplementedError` 约定** |
| A#2 裸 SQL 参数 | **删 `extra_where`/`extra_args`**（唯一用途"node NOT IN 失活"随变更 2/3 消失，改 `filter={'status':'运行'}`）；`order_by` 改**结构化** `[(字段,方向)]` |
| A#3 `register` | **全量 upsert**；`created_at` 首次保留由 **service 层**做（两后端共用） |
| A#4 `set_default` | **无 etcd 事务**，维持与 SQL 等价的非原子序列；**组合操作留 service 层** |
| 原则 | etcd 实现的接口与 SQL **行为等价**，契约测试守门 |
| status 语言 | **中文** `运行` / `停止` / `异常`（与代码一致） |

**B 组（环境 / 配置，均已与元戎确认）**

| # | 决定 |
|---|---|
| B#8 API 版本 | **`/v3`**（etcd **3.5.24**） |
| B#9 endpoint | **单 endpoint**（一体机单节点），无 failover |
| B#10 http/https | **由 endpoint scheme 决定**（元戎默认 `https`） |
| B#5 客户端证书 | **不强制**：与组件 mTLS 一致——配了 `ETCD_TLS_*` 走 mTLS、没配则不认证 |
| B#6 namespace | key 前缀、**无 RBAC 隔离**；用 `A2X_REGISTRY_ETCD_NAMESPACE`（我方配置），未配则**默认 `a2x-registry`** |
| B#7 配额 | **由元戎约束**，我方不实现配额 |

**C 组（设计决策）**

| # | 决定 |
|---|---|
| C#11 客户端库 | **urllib**（stdlib，与 rqlite 一致、零新依赖）打 `/v3/kv/*` |
| C#12 分页续传 | **不做** continuation token；全量取 + 应用层切片 |
| C#13 命名撞车 | `_` 前缀保留给 `_meta`；registry 名不以 `_` 开头（`create_registry` 加校验） |
| C#14 读一致性 | **linearizable**（etcd 默认，对齐 rqlite） |
| C#15 错误映射 | etcd 基础设施错误（超时 / 不可用 / 配额）→ **`502`** |
| C#16 启动失败 | etcd 不可用即 **fail-fast**（同 SQL） |
| C#17 模式限制 | **etcd 仅 appliance**：`DB_KIND=etcd` 要求 `MODE=appliance` |
| C#18 预置镜像 | 生产**暂无**预置；测试 seed 改走 `register()`（后端无关）；将来预置也走 `register()` |
| C#19 老数据迁移 | **v1 不做**自动迁移（etcd 全新起） |
| C#20 心跳独立性 | **已失效**——心跳随变更 2 移除，不涉及 etcd |

**D 组（测试 / 部署）**

| # | 决定 |
|---|---|
| D#21 CI etcd | CI 起 **Docker 真实 etcd**（`/v3` gateway），契约测试打真实实例、不 mock |
| D#22 契约测试范围 | 同一套契约测试**参数化跑 {SQL, etcd}**（SQL 侧取 sqlite / rqlite 之一） |
| D#23 断言 | **删** `NotImplementedError` 断言（etcd 全实现，A#1） |
| D#24 env | `DB_KIND=etcd` / `DB_ENDPOINT` / `ETCD_NAMESPACE`（默认 `a2x-registry`）/ 可选 `ETCD_TLS_CA/CERT/KEY` |
| D#25 ExecStartPre | 起 backend 前**探 etcd `/health` 就绪**（避免启动竞态，呼应 C#16） |

**已与元戎确认（3 条）**：① etcd **不强制客户端证书**——与组件 mTLS 一致，配了走 mTLS、没配不认证；② **不做 RBAC 隔离**，namespace 用我方配置（`A2X_REGISTRY_ETCD_NAMESPACE`），未配则默认 `a2x-registry`；③ 存储**由元戎约束**（我方不实现配额）。至此全部决策闭环，无待定项。
