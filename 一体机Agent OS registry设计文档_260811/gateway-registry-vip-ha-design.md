# Gateway 与注册中心 一主多备 + VIP 高可用设计方案与开发计划

> 适用范围：jiuwenswarm Gateway、A2X 注册中心

---

## 1. 背景与目标

当前 jiuwenswarm 的 Gateway 与 A2X 注册中心（下称“注册中心”）均以**单点 systemd/nohup** 方式部署在单台机器上，缺少高可用能力。业务要求提供：

- **一主多备 + VIP**：对外只暴露浮动 VIP，主节点故障时 VIP 漂移到备机，业务无感或低感切换。
- **注册中心与 Gateway 解耦**：两者不强制同机，各自分配独立 IP、独立 VIP。
- **网络隔离约束**：所有 VIP 必须绑定在 **lo 之外的第一个网卡**（内网网段）上；**只能通过 VIP 访问** gateway / 注册中心，禁止通过其它网卡（物理 IP / 其它网段）访问。

---

## 2. 现状分析

### 2.1 Gateway 部署流程

入口 [deploy.sh](/jiuwenswarm/deploy/yuanrong/deploy.sh) 的 `up gateway` → [gateway_handler.sh](/jiuwenswarm/deploy/yuanrong/gateway_handler.sh) 的 `deploy_gateway` → `gateway_deploy_process`，是**单机单实例**逻辑：

1. `gateway_resolve_host`：确定 master 节点（`MASTER_NODE_IP` 或 `CLUSTER_HOSTS[0]`）。
2. `gateway_gen_config`：渲染 `config.yaml` + 生成 `.env`。
3. 远程执行 `jiuwenswarm-init -f` 初始化工作区。
4. 下发 `config.yaml` / `.env` 到 `~/.jiuwenswarm/config/`（多实例为 `~/.jiuwenswarm-instances/<name>/config`）。
5. 启动：优先 systemd（unit + drop-in 注入 `User=`、`PATH`、`GATEWAY_HOST/PORT`、`WEB_PORT`），无 systemd 回退 nohup。
6. 健康检查。

关键点：所有 gateway 都部署到**同一个 master host**，只维护单一进程。改造目标就是扩展到“多节点各起一个进程 + VIP 绑定 active”。

### 2.2 注册中心（agent-registry）部署流程

参考 [agent-os/deploy/agent-gateway/module.sh](/agent-os/deploy/agent-gateway/module.sh)（目录名虽为 `agent-gateway`，实际是 A2X 注册中心；数据库后端：**单机临时 sqlite，后续改为 etcd**）。钩子函数：

| 钩子 | 作用 |
|------|------|
| `agent-gateway_install` | 安装 a2x-registry whl，准备运行环境，生成 systemd unit/drop-in（或 nohup 环境） |
| `agent-gateway_up` | 现算 BIND（`_agentregistry_bind`：`CLUSTER_HOSTS` 首个 IP，否则网卡首 IP），起注册中心，健康检查 |
| `agent-gateway_down` | 停注册中心 |
| `agent-gateway_uninstall` | 停服务 + 清理 + 卸包 |

要点与改造相关项：

- **端口**：`A2X_REGISTRY_PORT=4003`（健康检查 `curl http://<bind>:4003/api/images`）。**4001/4002 为旧 rqlite 端口，已不再使用（保留此说明仅为提示旧端口不占用）。**
- **数据目录**：注册中心数据库（当前 sqlite 临时后端）由配置决定，后续 etcd 后端接入组网公共 etcd。
- **监听地址**：`_agentregistry_bind()` 目前取 `CLUSTER_HOSTS` 首个 IP 或本机网卡首 IP —— 是**本机物理 IP**，改造后应改为 **注册中心 VIP**。

### 2.3 本地数据 / 状态盘点（数据同步分析的输入）

#### Gateway 侧（`~/.jiuwenswarm/`，多实例为 `~/.jiuwenswarm-instances/<name>/`）

| 数据 | 路径 | 类型 | 权威性 |
|------|------|------|--------|
| 配置 | `config/config.yaml` + `config/.env` | 静态 | 需多节点一致 |
| 定时任务 | `agent/home/cron_jobs.json` | 动态 | gateway 与 agentserver 共享写（[store.py](/jiuwenswarm/jiuwenswarm/gateway/cron/store.py#L117-L124)，`portalocker` 跨进程锁） |
| 会话映射 | `agent/.checkpoint/session_map.json` | 动态 | 缓存（[session_map.py](/jiuwenswarm/jiuwenswarm/gateway/routing/session_map.py#L49-L53)），可重建 |
| 附件（上传图片） | `agent/sessions/<sid>/uploads/` | 动态 | **唯一二进制副本**（见 §4.4） |
| 日志 | `agent/.logs/` | 动态 | 各节点独立即可 |

#### 注册中心侧

| 数据 | 路径 | 类型 | 权威性 |
|------|------|------|--------|
| 注册中心数据库 | 后端为 sqlite（单机临时）/ etcd（后续） | 动态 | 注册中心权威数据（镜像/Agent 注册信息） |
| 配置 | drop-in env | 静态 | 多节点一致 |

> 说明：注册中心是**基本静态的服务**，其数据由数据库后端承载。当前 **sqlite 为单机临时后端**；后续接入**组网公共 etcd**（分布式），注册中心数据天然具备多副本与高可用（见 §4.3）。

### 2.4 现有可复用资产

项目已提供一套 **NFS 共享工作空间脚本**（[scripts/nfs](/jiuwenswarm/scripts/nfs)）：`setup_nfs_server.sh` / `setup_nfs_client.sh` / teardown 系列，用于 Linux 节点间共享 `jiuwenswarm` 工作空间目录。**可作为附件共享存储的现成实现**（见 §4.4 遗留问题可选项）。

---

## 3. 总体架构设计

### 3.1 组网拓扑

内网网段示例：`192.168.1.0/24`。

#### 节点与 IP 清单（示例）

| 节点 | 内网物理 IP | 角色 | 服务 |
|------|-------------|------|------|
| gw-n1 | `192.168.1.101` | Gateway 主 | jiuwenswarm-gateway |
| gw-n2 | `192.168.1.102` | Gateway 备 | jiuwenswarm-gateway |
| reg-n1 | `192.168.1.201` | 注册中心主 | agent-registry（4003） |
| reg-n2 | `192.168.1.202` | 注册中心备 | agent-registry（4003） |
| etcd   | `192.168.1.203` | 公共组件 | etcd（注册中心数据库后端，后续）+ 附件元数据（可选） |

| VIP | 地址 | 绑定网卡 | 对应 keepalived 实例 | 对外端口 |
|-----|------|----------|----------------------|----------|
| Gateway VIP | `192.168.1.211` | 内网网卡 `eth-int` | VI_GATEWAY | 19000 / 19001 / 2223 |
| Registry VIP | `192.168.1.212` | 内网网卡 `eth-int` | VI_REGISTRY | 4003 |

> 说明：`192.168.1.0/24` 为主备与 VIP 规划示例；实际网段/网卡名以现场为准，但**所有 VIP 必须绑定到 lo 之外的第一个内网网卡**这一约束不变。

- **Gateway 与注册中心可不在同一台机器**，各自独立 IP、独立 VIP。
- 每个服务的“主/备”由各自的 keepalived `vrrp_instance` 独立管理，互不影响。

### 3.2 VIP 绑定规则（硬性约束）

- **绑定网卡**：所有 VIP 绑定到 **`lo` 之外的第一个网卡**（即内网网卡，示例 `eth-int`），**不绑 `lo`、不绑默认路由网卡之外的其他网卡**。
- **进程只 listen 在 VIP 上**：Gateway 的 `GATEWAY_HOST`/`WEB_HOST` 与注册中心的 BIND 都设为对应 VIP，**不使用 `0.0.0.0`**。
  - 理由：绑 `0.0.0.0` 会在所有网卡暴露，违反“只能通过 VIP 访问”约束。
- **兜底访问控制（可选加固）**：加 iptables 规则，仅允许目的地址为 VIP 的入站，其余网卡拒绝，双保险。

```
# keepalived virtual_ipaddress 示例（内网网卡 eth-int）
192.168.1.211  dev eth-int   # Gateway VIP
192.168.1.212  dev eth-int   # Registry VIP
```

### 3.3 keepalived 配置（每服务一个 vrrp_instance）

每个服务（gateway / 注册中心）各维护一个 `vrrp_instance`：

```text
global_defs {
    router_id HA_ROUTER
}

vrrp_script chk_gateway {
    script "/etc/keepalived/check_gateway.sh"   # 见 §3.4 冷启动/Warmup 处理
    interval 2
    fall 3          # 连续 3 次失败才判 down，覆盖冷启动窗口
    rise 2
}

vrrp_script chk_registry {
    script "/etc/keepalived/check_registry.sh"
    interval 2
    fall 3
    rise 2
}

vrrp_instance VI_GATEWAY {
    state BACKUP                   # 所有节点均 BACKUP + priority 区分主备（nopreempt 的前提）
    interface eth-int              # 内网网卡（lo 之外的第一个网卡）
    virtual_router_id 51
    priority 100                   # 主节点高，备节点低
    advert_int 1
    nopreempt                      # 见下方说明：避免恢复节点冷启动时期抢占造成抖动
    virtual_ipaddress { 192.168.1.211 dev eth-int }   # Gateway VIP
    track_script { chk_gateway }   # 探测 gateway 端口 19000/19001 + warmup 状态
    notify_master /etc/keepalived/gateway-notify.sh master
    notify_backup /etc/keepalived/gateway-notify.sh backup
    notify_fault   /etc/keepalived/gateway-notify.sh fault
}

vrrp_instance VI_REGISTRY {
    state BACKUP
    interface eth-int
    virtual_router_id 52
    priority 100
    advert_int 1                    # 注意：正确字段为 advert_int
    nopreempt                       # 禁止高优先级节点在其恢复后立刻抢占当前 master，仅主节点配置
    virtual_ipaddress { 192.168.1.212 dev eth-int }   # Registry VIP
    track_script { chk_registry }  # 探测注册中心端口 4003 + warmup 状态
    notify_master /etc/keepalived/registry-notify.sh master
    notify_backup /etc/keepalived/registry-notify.sh backup
    notify_fault   /etc/keepalived/registry-notify.sh fault
}
```


### 3.4 故障切换流程（服务与 VIP 联动）

因为服务进程 **bind 在浮动 VIP 地址** 上，VIP 漂移时进程必须同步启停：

1. keepalived `track_script` 持续探测本服务端口与 **warmup 状态**；主节点故障 → backup 升主。
2. 新主 `notify_master` → 启动/重启对应服务进程，使其重新 bind 到刚漂来的 VIP。
3. 旧主 `notify_backup` / `notify_fault` → 停止对应服务进程，避免旧 socket 悬空、端口冲突。
4. 客户端访问的 VIP 不变，配置无需改动。

#### 冷启动（Warmup）保护：避免误判 down 触发切换（低优先级）

服务冷启动阶段（进程刚拉起、依赖未就绪）若被健康探测直接判 down，会导致刚接管的节点被再次切换，造成抖动。设计如下：

- **服务暴露 `/api/warmup-status`**（gateway 与注册中心均需），返回状态：
  - `{"status": "ready"}`：服务已就绪，可对外服务。
  - `{"status": "warming"}`：仍处于冷启动/依赖就绪中，**即将就绪**。
  - `{"status": "error"}`：启动失败/不可恢复，应判 down。
- **`track_script` 判定逻辑**（`check_gateway.sh` / `check_registry.sh`）：
  | 探测结果 | track_script 处理 | 对 keepalived 的影响 |
  |----------|-------------------|----------------------|
  | `ready` | 判 up（exit 0） | 正常持有/竞争 VIP |
  | `warming` | **不计入失败**（返回"过渡态"，不增加 `fall` 计数） | **不触发切换**，VIP 保持当前节点 |
  | `error` / 端口不可达 | 判 down（exit 非 0） | 累加 `fall`，连续 N 次后切换 |
  | 端口未开（进程未监听到） | 判 down | 累加 `fall` |
- **配套手段**（多级兜底，避免冷启动误切）：
  1. `track_script` 的 `fall N`（如 `fall 3`）——慢启动在 N 个探测周期内不会触发切换。
  2. `warming` 状态不计入失败——即使进程已监听但未 ready，也不判 down。
  3. `nopreempt`（§3.3）——恢复节点不会在 warmup 完成前抢占。
- **实现约定**：
  - 服务在 `ready` 前不对外正常应答业务请求，但 `/api/warmup-status` 必须始终可访问（监听端口即开）。
  - 也可在 `notify_master` 中先启动服务、等待 `/api/warmup-status = ready` 后再认为本机稳定持有 VIP；期间 `track_script` 保持返回 `warming`。

### 3.5 跨机引用

- Gateway 配置 `gateway.agentos.registry.endpoint` 指向 **Registry VIP `192.168.1.212`**（[gateway-config-yuanrong.template.yaml#L480](/jiuwenswarm/deploy/yuanrong/conf/gateway-config-yuanrong.template.yaml#L480)），而非单节点 IP。
- `gateway.agentos.registry.node` 使用 **gateway 所在节点的内网物理 IP**（如 `192.168.1.101`/`192.168.1.102`，用于节点心跳上报）。

---

## 4. 数据同步设计（重点）

### 4.1 数据分类矩阵

| 数据 | 所在组件 | 是否需跨节点同步 | 同步策略 |
|------|----------|------------------|----------|
| `config.yaml` / `.env` | Gateway | **必须一致** | 部署下发到所有节点（静态） |
| 注册中心 drop-in env | 注册中心 | **必须一致** | 部署下发（静态） |
| `session_map.json` | Gateway | 可不同步 | 缓存，可重建 |
| `cron_jobs.json` | Gateway | 建议仅主节点跑 cron | 主备一致或主节点独占 |
| 附件 `uploads/` | Gateway | **遗留问题** | 见 §4.4 |
| 注册中心数据库（sqlite/etcd） | 注册中心 | etcd 天然多副本；sqlite 为单机临时 | 见 §4.3 |
| 日志 | 两者 | 否 | 各节点独立 |

### 4.2 Gateway 数据同步策略

- **配置**：同一份渲染产物下发到所有节点，保证备机接管时行为一致。
- **`session_map.json`（缓存）**：丢失仅影响“会话连续性”（同一身份复用 session_id），下次由 AgentServer `session.create` 重建，**不丢数据**。可随部署周期同步，也可不同步。
- **`cron_jobs.json`（动态）**：为避免主备双写冲突，建议 **仅主节点启用 cron 调度**；备机接管后再加载同一份 `cron_jobs.json`（随部署/接管时同步）。若追求无缝，可走共享存储（见 §4.4）。

### 4.3 注册中心数据同步策略

注册中心数据库后端演进：**当前单机 sqlite（临时）→ 后续组网公共 etcd（分布式）**。同步策略按下述两种情况分治：

- **等后端为 etcd（目标形态）**：注册中心数据写入**组网公共 etcd**，etcd 本身是分布式强一致存储，天然多副本、多点可读。此时注册中心**无需额外的数据同步**，备机接管后直接读写同一 etcd 即可，真正无缝切换。这是本方案建议的最终形态。
- **当前 sqlite 过渡形态**：注册中心数据为**单机本地副本**。备机不接流量时，可随部署/接管时从主节点冷同步，或接受接管后短暂回退（与 Gateway 附件遗留问题类似，但此处数据为注册信息，丢失影响更大，应优先切换到 etcd）。

> 建议：注册中心侧优先推进 **etcd 后端落地**，可同时消除“单机临时 sqlite”的数据单点风险，并让 VIP 主备切换对注册中心数据完全透明。

### 4.4 遗留问题：附件（上传图片）跨节点同步

#### 问题本质

附件由 Gateway [media_attachments.py](/jiuwenswarm/jiuwenswarm/gateway/media_attachments.py#L88-L99) 落盘到 **本机** `~/.jiuwenswarm/agent/sessions/<sid>/uploads/`，消息里只保留 `path`；AgentServer 侧 [user_prompt_builder.py](/jiuwenswarm/jiuwenswarm/agents/harness/common/prompt/user_prompt_builder.py#L44-L72) 从该 `path` 读**真实图片字节**。因此图片是**单机唯一二进制副本**，且 gateway 与 AgentServer 需访问同一文件系统。当前组网**没有分布式/共享存储池**（仓库中唯一 MinIO 属于可观测性栈，与附件无关）。

#### 业务影响（若不做同步）

| 场景 | 是否有影响 | 说明 |
|------|-----------|------|
| 主节点正常、无切换 | 无 | 附件本地读写正常 |
| 切换窗口内进行中的多模态对话（用户刚上传图片，主节点随即故障） | **有，低-中** | 备机接管后该 `path` 读不到，这轮基于图片的对话上下文丢失 |
| 已完成的对话 / 历史会话 | 无 | 会话内容在 AgentServer，gateway 无权威状态 |
| 切换后新上传的附件 | 无 | 落备机本地，正常工作 |

**结论**：附件不同步仅影响“切换瞬间正在进行的多模态对话”，属即时性损失，不涉及持久数据，**可接受的遗留问题**，不阻塞 VIP 方案主线落地。

#### 可选处置（后续演进方向）

1. **共享存储（推荐，现成脚本可复用）**：用项目自带 [scripts/nfs](/jiuwenswarm/scripts/nfs) 将 `~/.jiuwenswarm/agent/sessions` 挂载到所有 gateway/AgentServer 节点，附件天然多节点可见。`CronJobStore` 已用 `portalocker` 文件锁，兼容多进程并发写。
2. **对象存储 + etcd 元数据**：附件字节放对象存储（S3/MinIO/OBS）或共享盘；**etcd 只存 `session_id → 附件 key/元数据` 索引**（etcd 适合小值，不适合作附件二进制存储，附件单张上限 10MB、一次最多 8 张，远超 etcd 合理阈值）。

### 4.5 数据同步场景清单（完整）

| # | 场景 | 预期行为 | 依赖 |
|---|------|----------|------|
| S1 | 正常部署（主节点） | 配置下发、服务启动、VIP 绑定 active | 部署脚本 |
| S2 | 正常部署（备节点） | 配置下发、服务待命、不接流量 | 部署脚本 |
| S3 | 主 Gateway 故障 | VIP 漂到备机 + 备机 gateway 启动 | keepalived + notify |
| S4 | 主注册中心故障 | Registry VIP 漂到备机 + 备机注册中心启动 | keepalived + notify |
| S5 | Gateway 与注册中心不同机 | Gateway 经 Registry VIP 访问注册中心，互不感知切换 | 跨机引用配置 |
| S6 | 原主恢复 | 加 `nopreempt`：不自动回切，VIP 保持当前 master；不加则依赖 warmup 探测自动回切 | keepalived 策略（§3.3） |
| S7 | 切换窗口内进行中的多模态对话 | 附件可能读不到（遗留问题） | 见 §4.4 |
| S8 | 切换后新会话/新附件 | 正常 | — |
| S9 | 仅主节点跑 cron | 备机接管后加载 cron_jobs.json | 同步策略 |
| S10 | 多实例（`JIUWENSWARM_INSTANCE_NAME`） | VIP/服务名按实例区分 | 部署脚本 |
| S11 | 服务冷启动（接管/重启） | `/api/warmup-status=warming` 不计入失败，不触发切换；`ready` 后才对外服务 | warmup 接口 + track_script（§3.4） |
| S12 | 服务启动失败/不可恢复 | `/api/warmup-status=error` 判 down，累加 `fall` 后切换 | warmup 接口 + track_script（§3.4） |

---

## 5. 部署 / 开发计划

按阶段推进；每阶段含改动范围、涉及文件、验收标准。

### 阶段 0：环境与预研（前置）
- 确认内网网卡名称（lo 外第一个网卡）与网段、VIP 网段规划。
- 确认 keepalived 是否已部署（未部署则补装）。
- 确认注册中心当前是否单点部署、数据库后端（sqlite/etcd）与规模。
- **产出**：内网/VIP/节点 IP 清单、网卡名确认。

### 阶段 1：Gateway 多节点化部署
- 改动：[gateway_handler.sh](/jiuwenswarm/deploy/yuanrong/gateway_handler.sh) 的 `gateway_deploy_process` 从“只部署 master host”改为遍历 `GATEWAY_HOSTS` 全部节点逐台部署。
- 改动：`GATEWAY_HOST`/`WEB_HOST` 绑定改为 **Gateway VIP**（不再用 `0.0.0.0` 或 master 物理 IP）。
- 改动：systemd 服务名按节点/实例区分，避免跨节点冲突。
- 改动：`gateway.agentos.registry.endpoint` 指向 Registry VIP。
- 验收：所有节点可独立拉起 gateway；仅在 Gateway VIP 可访问，其它网卡不可达。

### 阶段 2：注册中心多节点化部署
- 依据 [module.sh](/agent-os/deploy/agent-gateway/module.sh) 钩子扩展：
  - `agent-gateway_install`：支持多节点安装（复用现有逻辑）。
  - `agent-gateway_up`：BIND 改为 **Registry VIP**（改造 `_agentregistry_bind`），健康检查不变。
- 改动：`agentos.registry.node` 使用节点内网物理 IP。
- 验收：注册中心仅在 Registry VIP 可访问；`/api/images` 健康检查通过。

### 阶段 3：keepalived 双实例 + notify 联动 + Warmup 保护
- 新增 keepalived 配置模板（Gateway VI + Registry VI，绑内网网卡，含 `nopreempt` 决策）。
- 新增 notify 脚本（master→启动服务 / backup·fault→停止服务，绑定 VIP）。
- 新增 `track_script` 端口探测（gateway 19000/19001；registry 4003）。
- **新增 `/api/warmup-status` 接口**（gateway 与注册中心）并接入 `track_script`（`warming` 不计失败，`error`/不可达判 down），避免冷启动误切。
- 可选加固：iptables 仅放行 VIP 入站。
- 验收：主节点故障 → VIP 漂移 → 备机服务自动接管；恢复后按策略回切；冷启动不触发误切换（S11/S12）。

### 阶段 4：数据同步策略落地
- `session_map.json`：随部署同步（可选）。
- `cron_jobs.json`：仅主节点跑 cron，接管时同步（或共享存储）。
- **遗留问题确认**：附件不做同步，记录为已知限制（§4.4）。
- 验收：S1–S12 场景逐一验证。

### 阶段 5（可选增强）：注册中心 etcd 后端 + 附件共享存储
- 将注册中心数据库后端从 sqlite 迁移/接入**组网公共 etcd**，消除注册中心数据单点。
- 用 [scripts/nfs](/jiuwenswarm/scripts/nfs) 或对象存储 + etcd 元数据解决附件同步，消除遗留问题。
- 验收：注册中心数据多副本（etcd）；附件跨网关/AgentServer 节点可见。

### 里程碑与交付物
- 1（阶段 1–3）：Gateway + 注册中心 VIP 主备可切换。
- 2（阶段 4）：数据同步策略落地，遗留问题明确。
- 3（阶段 5，可选）：注册中心 etcd 后端 + 附件共享，消除遗留问题。

---

## 6. 风险与注意事项

1. **VIP 绑定网卡**：必须确认“lo 外第一个网卡”是内网网卡，绑定错误会导致 VIP 不可达或暴露到错误网段。
2. **进程只绑 VIP**：服务 bind 到浮动地址，VIP 漂移瞬间进程需重启（notify 联动），存在秒级中断，属预期。
3. **注册中心数据库单点限制**：当前 sqlite（单机临时）下数据为单副本，接管靠冷同步/回退；接入组网公共 etcd 后即消除（见 §4.3）。
4. **cron 双写**：备机若同时跑 cron 会与主节点写 `cron_jobs.json` 冲突，务必“仅主节点跑 cron”。
5. **附件是唯一副本**：不共享时，切换窗口内多模态对话可能丢图（已记录为遗留问题）。
6. **目录命名**：注册中心代码目录为 `agent-gateway`，文档/实施中须统一理解其实际为注册中心，避免与 jiuwenswarm Gateway 混淆。
