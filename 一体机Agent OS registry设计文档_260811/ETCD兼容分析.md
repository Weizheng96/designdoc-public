# ETCD 兼容分析

> 目标：rqlite 与 etcd 双支持（依赖倒置抽象），etcd 复用元戎 etcd、仅一体机可选、只实现一体机所需接口。**约定：etcd 未实现的方法（分页/筛选/预约等）统一 `raise NotImplementedError`（英文提示），不返 None。** 基于 `feature/Agentregistry-dev` 当前实现。

## 1. 当前 SQL 调用接口 · 一体机涉及哪些

所有 SQL 最终经 `common/db.py` 的 `Backend.execute(sql)` / `query(sql)`；上层 `register/service.py::RegistryTableService` 把它们封成方法。下表列这些方法（= SQL 接口），标注一体机（image / instance）是否用：

| 方法 | 作用 | SQL | 一体机 |
|---|---|---|:--:|
| `create_registry(name, kind)` | 建命名注册表（元数据） | `INSERT registry_meta` | ✅ 建 images / instances |
| `get_kind` / `list_registries` | 查注册表 kind | `SELECT registry_meta` | ✅ 内部 |
| `register(name, entry)` | upsert 一行 | `INSERT … ON CONFLICT` | ✅ |
| `get(name, sid)` | 取一行 | `SELECT * WHERE pk` | ✅ 内部（patch/deregister 校验） |
| `patch(name, sid, fields)` | 改字段 | `UPDATE … WHERE pk` | ✅ |
| `deregister(name, sid)` | 删一行 | `DELETE WHERE pk` | ✅ |
| `query(name, filter)` | 按等值 filter 列 | `SELECT * WHERE filter` | ✅ |
| `query_paginated(…)` | 分页 + 排序 + 计数 | `SELECT … ORDER/LIMIT` + `COUNT` | ✅（但一体机不用分页/筛选） |
| `list_services` / `get_taxonomy_state` / `get_vector_config` / `get_register_config` / `get_lease_config` / `list_datasets_with_counts` / `delete_dataset` / `update_service` … | 数据集 / 服务 / 分类树 / 向量 / 租约（A2X 通用模式） | 各类 SQL | ❌ 非一体机 |

**→ 一体机只用 8 个方法**：`create_registry` / `get_kind` / `list_registries` / `register` / `get` / `patch` / `deregister` / `query`（外加 `query_paginated`，但其分页/筛选一体机不用）。均为**按主键 `(registry, service_id)` 的行级 CRUD + 前缀列举**，无 JOIN、无复杂查询。

## 2. 一体机接口抽象成哪些接口

把上面 8 个方法收敛为一个 **`TableRepo` 抽象接口**（依赖倒置的约定）。**直接沿用现有方法名**，使 `RegistryTableService`（rqlite）天然满足；新增 `EtcdTableRepo` 实现同一接口。image / instance 只依赖 `TableRepo`：

| 抽象方法（= 现有方法名） | 语义 |
|---|---|
| `create_registry(name, kind)` | 声明命名注册表 |
| `get_kind(name)` | 查注册表 kind（不存在返 None） |
| `register(name, entry)` | upsert 一行（幂等；`entry` 含 `service_id`） |
| `get(name, service_id)` | 取一行 / None |
| `patch(name, service_id, fields)` | 改部分字段 |
| `deregister(name, service_id)` | 删一行 |
| `query(name, filter=None)` | 按等值 filter 列（`filter=None` 返全部） |
| `query_paginated(…)` → **etcd 未实现即 raise** | 分页/排序/筛选（一体机不用，etcd 不实现） |

即抽象接口 = 这 8 个现有方法；后端不支持的（如 etcd 的 `query_paginated`）统一 **`raise NotImplementedError`（英文提示）**，不返 None。

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
│   ├── etcd_client.py   [新增] 子模块：etcd KV 连接封装
│   └── etcd_repo.py     [新增] 子模块：EtcdTableRepo（实现 TableRepo）
├── image/  └ service.py       子模块：镜像服务，依赖 _table_svc
├── instance/ └ service.py     子模块：实例服务，依赖 _table_svc
└── backend/                   模块：装配与启动
    ├── __main__.py            子模块：读 env（DB_KIND / DB_ENDPOINT / ETCD_NAMESPACE）
    └── startup.py             子模块：工厂选 repo 注入 image/instance service
```

依赖方向：`image/instance` → `TableRepo`（抽象）← `RegistryTableService`(rqlite) / `EtcdTableRepo`(etcd)；`backend` 工厂按 `A2X_REGISTRY_DB_KIND` 选注入。键布局：`/{namespace}/{registry}/{service_id}` → 行 JSON（`namespace` 来自元戎，env 配）。两步走：**3.1 先抽接口 + rqlite 归位**（零行为改、可回归）→ **3.2 加 etcd**（按 `DB_KIND` 灰度）。

> **元戎 etcd 已核实（定实现方向）**：元戎自带并启动 etcd、grpc-gateway 开着——**用 httpx 打 `/v3/kv/put|range|deleterange|txn`**（key/value base64，与元戎自带冒烟测试同款），端点默认 `https://<ip>:32379`，compaction/配额由元戎管。**本次两条决策**：① 连接安全 **配了证书走 mTLS、没配不认证**（与组件 mTLS 同一开关）；② namespace **先按 key 前缀约定**（客户端给 key 加 `{namespace}/` 前缀），暂不依赖 RBAC。待元戎最终确认的仅剩：一体机 etcd 是否强制客户端证书、给我们的前缀名与配额。

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
    def query_paginated(self, name: str, filter=None, order_by="", limit=-1,
                        offset=0) -> Tuple[List[dict], int]: ...  # etcd 未实现则 raise NotImplementedError
```

各方法功能：
- `create_registry(name, kind)`：声明一个命名注册表（如 `images` / `instances`）及其 kind，启动时建；已存在则幂等无副作用。
- `get_kind(name)`：返回命名注册表的 kind（`image` / `instance` / …），不存在返 None——供其余方法定位物理表 / 键前缀。
- `register(name, entry)`：**upsert 一行**（`entry` 含 `service_id`）；同 `service_id` 覆盖、`created_at` 保留首次值；返回存后的行。
- `get(name, service_id)`：按主键 `(name, service_id)` 取一行，不存在返 None。
- `patch(name, service_id, fields)`：**部分更新** `fields` 指定的列；条目不存在报错；返回更新后的行。
- `deregister(name, service_id)`：删一行，返回是否删除（幂等：本不存在返 False）。
- `query(name, filter)`：按**等值 filter** 返回匹配行列表；`filter=None` 返回该注册表全部行。
- `query_paginated(name, filter, order_by, limit, offset)`：排序 + 分页，返回 `(行列表, 过滤后总数)`；**后端不支持时 `raise NotImplementedError`（英文，如 etcd）**。

行数 **~40–60**。

#### 3.1.2 `register/service.py` — rqlite 实现归位
`RegistryTableService` 已实现上述 8 方法。仅声明其满足 `TableRepo`（`class RegistryTableService(TableRepo)` 或结构化类型），补类型标注；`query_paginated` 返回类型对齐为可空。**零逻辑改**。行数 **~0–20**。

#### 3.1.3 `image/service.py` · `instance/service.py` — 上层依赖抽象
把 `_table_svc` 的类型标注由具体类改为 `TableRepo`（依赖倒置：上层只依赖接口）。调用点不变。行数 **~5–10**。

#### 3.1.4 `backend/startup.py` — 装配工厂
把"建 rqlite Backend → 造 RegistryTableService → 注入 service"抽成 `build_table_repo(cfg) -> TableRepo` 工厂函数（现状即此逻辑，仅归拢）。行数 **~10–20**。

> 小计 **~55–110 行**，无行为变化，跑现有测试回归即可。

### 3.2 对接 ETCD

#### 3.2.1 `register/etcd_client.py` **[新增]** — KV 连接封装
用 **httpx 直连 etcd v3 HTTP/JSON gateway**（`/v3/kv/*`，key/value 走 base64；与元戎自带冒烟测试 `etcd_put/etcd_delete` 同款，免 gRPC 依赖）。**TLS 配置驱动**：配了客户端证书走 mTLS，没配则不认证；所有 key 统一加 `namespace/` 前缀（**key 前缀约定隔离**）。

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
- `__init__`：建 httpx client 连 `endpoint`（默认 `https://<ip>:32379`）；**配了 `cert`/`key`/`ca` → mTLS（httpx `cert=(cert,key)` + `verify=ca`），没配 → 明文/不验**；所有 key 自动加 `namespace/` 前缀。
- `get(k)`：`POST /v3/kv/range`（单键），返回值 JSON 或 None。
- `create(k, v)`：`POST /v3/kv/txn`，`compare create_revision==0` → **仅不存在时写**（幂等），已存在返 False。
- `put(k, v, mod_revision)`：`POST /v3/kv/put`；给了 `mod_revision` 则用 txn 做 **CAS 乐观锁**。
- `delete(k)`：`POST /v3/kv/deleterange`（单键）。
- `range(prefix)`：`POST /v3/kv/range`（`range_end` = 前缀尾字节 +1），返回前缀下所有值 JSON——供按 registry 列举。

行数 **~50–70**（含 base64 编解码 + TLS 装配）。

#### 3.2.2 `register/etcd_repo.py` **[新增]** — EtcdTableRepo
`class EtcdTableRepo(TableRepo)`，用 `EtcdClient` 实现 8 方法；键 `/{ns}/{registry}/{service_id}`：

- `register`=`create`/`put`(幂等)、`get`=`get`、`patch`=get+合并+put(`mod_revision`)、`deregister`=`delete`、`query`=`range(prefix)`(等值 filter 客户端过滤，不支持则忽略)。
- `create_registry` / `get_kind`=元数据键 `/{ns}/_meta/{registry}`。
- **`query_paginated`（及未实现的筛选）→ `raise NotImplementedError`**（英文，如 `"pagination not supported by etcd backend"`；一体机不用分页）。

行数 **~150–220**。

#### 3.2.3 `backend/__main__.py` — 配置
`DB_KIND` 枚举加 `etcd`；复用 `A2X_REGISTRY_DB_ENDPOINT`（etcd 时必填）；新增 `A2X_REGISTRY_ETCD_NAMESPACE`（key 前缀）+ `A2X_REGISTRY_ETCD_TLS_CA/CERT/KEY`（**三者配齐即对 etcd 走 mTLS，没配则不认证**，与组件 mTLS 同一"配了才开"的开关思路）；校验。行数 **~20–30**。

#### 3.2.4 `backend/startup.py` — 工厂分支
`build_table_repo`（3.1.4）加 `DB_KIND=etcd` 分支 → 造 `EtcdTableRepo`；etcd 时不建 rqlite 连接。行数 **~15–25**。

#### 3.2.5 `pyproject.toml` — 依赖
如走 httpx 直连则免；若用 etcd3 库则加可选 extra `etcd`。行数 **~5**。

#### 3.2.6 agent-os `deploy/agent-gateway/module.sh` · `build/build.sh` — 部署
etcd 模式：不装 rqlite rpm、不起 rqlited；注入 `DB_KIND=etcd`+endpoint+namespace 指向元戎 etcd（沿用 mTLS 那套 env 注入 + `_agentgw_has_systemd` 式开关）。行数 **~20–40**。

#### 3.2.7 `tests/db/test_table_repo_etcd.py` **[新增]** — 契约测试
8 方法在 rqlite vs etcd 行为一致（etcd 未实现项断言 `raise NotImplementedError`）；etcd 用自建实例做真实接口测试。行数 **~100–180**。

> 小计 **~345–555 行**。

**合计（3.1 + 3.2）≈ 400–665 行**：新增 4 个 `.py`（`table_repo` / `etcd_client` / `etcd_repo` / 测试），改动 5 个现有子模块（`register/service`、`image/service`、`instance/service`、`backend/__main__`、`backend/startup`）+ 2 个部署脚本。etcd 硬限（单值 1.5MB / txn 128 op）对一体机小数据无碍。
