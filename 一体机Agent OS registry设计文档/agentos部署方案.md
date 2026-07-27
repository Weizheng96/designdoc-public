# agentos 部署注册中心方案

> 说明 A2X 注册中心如何接入一体机 **agentos** 统一部署框架。非注册中心组件（yuanrong / jiuwenswarm / jiuwenbox）从略。

## 1. agentos 整体工作流程

agentos 分**构建**和**部署**两个阶段：一个造包，一个用包。

```
build.sh（构建机，联网）                       agentos.sh（目标机）
  下载各组件 wheel ──打包──▶ AgentOS-Server.tgz ──解压──▶ install → up
```

### 构建阶段（`build/build.sh`，在联网的构建机上跑）

- **第一步**：解析参数、按模式（`daily` / `release`）确定各组件的下载地址与文件清单。**【需要修改】** 把注册中心 wheel（`REGISTRY_WHL_URL`）和 rqlite rpm（`RQLITE_RPM_URL`）的下载地址加入清单。
- **第二步**：从 OBS / gitcode 下载各组件的 wheel（openyuanrong、jiuwenswarm 等）。**【需要修改】** 增加下载注册中心 wheel 和 rqlite rpm。
- **第三步**：把下载的 wheel 连同仓库的 `deploy/` 目录，打包成 `AgentOS-Server-<arch>.tgz`。**【需要修改】** 打包时纳入注册中心 wheel 和 rqlite rpm。

> 这样注册中心和其它组件一致：构建时下载、打进包，部署时从本地 `AGENTOS_ROOT` 安装。

### 部署阶段（`deploy/agentos.sh`，在目标机上跑）

- **第一步**：把 `AgentOS-Server-<arch>.tgz` 传到目标机并解压，得到 wheel 包和 `deploy/` 目录。
- **第二步**：`bash agentos.sh install` —— 按 `MODULES` 声明顺序，逐个模块本机安装各自的包（只装包，不起服务）。**【需要修改】** 新增 agent-gateway 模块的 `install` 钩子（从 `AGENTOS_ROOT` 装注册中心 whl + rqlite rpm，并放置 systemd unit 与配置）。
- **第三步**：`bash agentos.sh up` —— 按 `MODULES` 声明顺序，逐个模块启动服务。**【需要修改】** 新增 agent-gateway 模块的 `up` 钩子（用 systemd 起 rqlite + 注册中心 backend）。
- **（可选）** `down` 停止并卸载、`restart` 重启、`uninstall` 卸载 whl；`down` / `uninstall` 按声明的**逆序**执行。**【需要修改】** agent-gateway 模块的 `down` / `uninstall` 钩子。

---

## 2. 注册中心修改内容

注册中心以 `agent-gateway` 模块接入。存储用 **rqlite**（第三方，rpm 安装），由本模块一并管理；注册中心 backend 连本机 rqlite（`http://127.0.0.1:4001`）。共三处改动。

### 2.1 配置 `build/build.sh`（下载 wheel + rqlite rpm，打进包）

- 顶部加两个下载地址：
  - `REGISTRY_WHL_URL` —— 注册中心 wheel（`a2x_registry-*-py3-none-any.whl`，**架构无关**，daily/release 同源）。
  - `RQLITE_RPM_URL` —— rqlite 官方 **rpm**（**分架构**，按 `ARCH` 选 `x86_64` / `aarch64`）。
- 新增 `build_agent_gateway()`：下载上面两个文件到 `downloads/agent-gateway/`。
- `main()` 中调用 `build_agent_gateway`。
- `pack()` 中把 wheel 和 rpm 拷进 server 打包目录，随 `AgentOS-Server-<arch>.tgz` 分发。

> rqlite rpm 分架构，注意 `ARCH` 用 `uname -m` 的 `x86_64`/`aarch64`（rpm 的 arch 标签正是这两个，无需像二进制那样映射成 amd64/arm64）。

### 2.2 配置 `deploy/agentos.sh`（注册模块）

`MODULES` 数组加入 `"agent-gateway"`，位于 yuanrong 之后、jiuwenswarm 之前（gateway 运行期要写注册中心，故注册中心先起）：

```bash
MODULES=("jiuwenbox" "yuanrong" "agent-gateway" "jiuwenswarm")
```

调度逻辑不动。

### 2.3 实现 `deploy/agent-gateway/module.sh`（4 个钩子）

两个服务各由 **systemd** 托管：`rqlited.service`（存储，rqlite rpm 自带）+ `agent-registry.service`（注册中心，`install` 钩子用 heredoc 生成）。注册中心 unit 声明 `After=rqlited.service` / `Requires=rqlited.service`，启动顺序交给 systemd。

> **配置方式**：注册中心 backend **不读 `registry.env` 配置文件**，配置一律走环境变量。故 `up` 钩子**在脚本中现算 BIND、现设 `A2X_REGISTRY_*` 环境变量并注入服务**（下方 2.3.2），而非维护一个 registry.env 文件。各钩子按需取用 agentos 全局量（`AGENTOS_ROOT`、`YR_PYTHON_VERSION`、`CLUSTER_HOSTS`）。

`agent-registry.service` 关键字段：

```ini
[Unit]
Requires=rqlited.service
After=rqlited.service
[Service]
# 环境变量由 up 钩子现算现设并注入（脚本生成 systemd drop-in，非 registry.env 文件）
ExecStart=/usr/bin/python3.11 -m a2x_registry.backend  # install 用 command -v 解析绝对路径
Restart=on-failure                                   # 异常退出自动重启
RestartSec=3                                         # 重启前退避 3s
StartLimitIntervalSec=60                             # 熔断：60s 窗口内
StartLimitBurst=5                                    #   最多重启 5 次，超了停手（防疯狂重启）
[Install]
WantedBy=multi-user.target
```

> rqlite 的 rpm 自带 unit 通常已含 `Restart=on-failure`，如无则同样补上。

#### 2.3.1 `agent-gateway_install`

- **输入**：`AGENTOS_ROOT`（whl / rpm 所在目录）、`YR_PYTHON_VERSION`。
- **流程**：① 从 `AGENTOS_ROOT` 装 rqlite rpm（`dnf install -y ./rqlite-*.rpm`，**系统级**，含 `rqlited.service`）；② 系统级装注册中心 wheel（`pip install ./a2x_registry-*.whl`）；③ heredoc **生成** `agent-registry.service` 到 `/etc/systemd/system/`（`ExecStart` 用 `command -v python${YR_PYTHON_VERSION}` 解析到的绝对路径）；④ `systemctl daemon-reload`。
- **默认配置**：安装源恒为 `AGENTOS_ROOT` 本地文件（离线）；unit 生成在 `/etc/systemd/system/`。运行配置不在此步，改由 `up` 钩子现设环境变量（见 2.3.2）。

#### 2.3.2 `agent-gateway_up`

- **输入**：`CLUSTER_HOSTS`（agentos 透传的 `--hosts`）；未给则自动探测本机 IP。
- **流程**：
  0. **已运行则跳过**：`systemctl is-active` 检测到 `agent-registry` 已在跑则告警返回（需先 `down` 再重启，对齐 yuanrong/jiuwenbox）。
  1. **解析监听地址 `BIND`**：`--hosts` 给了 → 取第一个 IP（`CLUSTER_HOSTS` 首个，如 `./agentos.sh up --hosts 192.168.1.1,192.168.1.2` → `192.168.1.1`）；未给 → 同其它模块，用 `hostname -I` 的**第一个非 `127.0.0.1`** IP。
  2. **在脚本中现设环境变量**：把 `BIND` 与默认 `A2X_REGISTRY_PORT` / `MODE` / `DB_KIND` / `DB_ENDPOINT` 组成 `A2X_REGISTRY_*` **环境变量注入 systemd 服务**（脚本生成 drop-in，**不写 registry.env 文件**）。
  3. `systemctl enable --now rqlited agent-registry`——因 `Requires=rqlited.service`，systemd **先起 rqlited** 再起注册中心，`enable` 使二者开机自启。rqlited 用 rpm 默认（`localhost:4001`），无需覆盖。
  4. curl 健康检查 `GET /api/datasets` 就绪后返回（`systemctl start` 不等 uvicorn 真就绪，故仍需探测）。
- **默认端口**（两个服务，均与其它模块**不冲突**）：

  | 服务 | 端口 | 默认绑定 | 备注 |
  |------|------|---------|------|
  | 注册中心 backend | **4003** | 本机网卡 IP | 对外，gateway / 用户访问 |
  | rqlite HTTP | **4001** | `127.0.0.1` | 仅本机 backend 连 |
  | rqlite Raft | **4002** | `127.0.0.1` | 单机自身；多节点用（后续） |

  > 现有占用：yuanrong `31182`/`8888`、jiuwenswarm `8888`。注册中心相关服务归到 `400x` 段（`4001`/`4002`/`4003`），均空闲、不冲突。

- **默认配置**（由 `up` 钩子在脚本中设置为环境变量，backend 无 `--host`、不读配置文件）：

  | 变量 | 默认 | 说明 |
  |------|------|------|
  | `A2X_REGISTRY_BIND` | `--hosts` 首个 IP；未给则 `hostname -I` 首个非 `127.0.0.1` | 禁 `0.0.0.0` |
  | `A2X_REGISTRY_PORT` | `4003` | backend 端口 |
  | `A2X_REGISTRY_MODE` | `appliance` | 建镜像/实例表 |
  | `A2X_REGISTRY_DB_KIND` | `rqlite` | 存储后端 |
  | `A2X_REGISTRY_DB_ENDPOINT` | `http://127.0.0.1:4001` | 连本机 rqlite |

#### 2.3.3 `agent-gateway_down`

- **输入**：无。
- **流程**：`systemctl disable --now agent-registry rqlited`（停并取消自启，两个服务）。systemd 依赖关系保证注册中心先于 rqlited 停。
- **默认配置**：无。

#### 2.3.4 `agent-gateway_uninstall`

- **输入**：`YR_PYTHON_VERSION`。
- **流程**：① `systemctl disable --now agent-registry`；② 删 unit 与 drop-in 目录（`agent-registry.service` + `agent-registry.service.d/`）+ `systemctl daemon-reload`；③ `pip uninstall -y a2x-registry`；④ `rpm -e rqlite`。
- **默认配置**：无。
