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
- **第二步**：`bash agentos.sh install` —— 按 `MODULES` 声明顺序，逐个模块本机安装各自的包（只装包，不起服务）。**【需要修改】** 新增 agentregistry 模块的 `install` 钩子（从 `AGENTOS_ROOT` 装注册中心 whl + rqlite rpm，并放置 systemd unit 与配置）。
- **第三步**：`bash agentos.sh up` —— 按 `MODULES` 声明顺序，逐个模块启动服务。**【需要修改】** 新增 agentregistry 模块的 `up` 钩子（用 systemd 起 rqlite + 注册中心 backend）。
- **（可选）** `down` 停止并卸载、`restart` 重启、`uninstall` 卸载 whl；`down` / `uninstall` 按声明的**逆序**执行。**【需要修改】** agentregistry 模块的 `down` / `uninstall` 钩子。

---

## 2. 注册中心修改内容

注册中心以 `agentregistry` 模块接入。存储用 **rqlite**（第三方，rpm 安装），由本模块一并管理；注册中心 backend 连本机 rqlite（`http://127.0.0.1:4001`）。共三处改动。

### 2.1 配置 `build/build.sh`（下载 wheel + rqlite rpm，打进包）

- 顶部加两个下载地址：
  - `REGISTRY_WHL_URL` —— 注册中心 wheel（`a2x_registry-*-py3-none-any.whl`，**架构无关**，daily/release 同源）。
  - `RQLITE_RPM_URL` —— rqlite 官方 **rpm**（**分架构**，按 `ARCH` 选 `x86_64` / `aarch64`）。
- 新增 `build_agentregistry()`：下载上面两个文件到 `downloads/agentregistry/`。
- `main()` 中调用 `build_agentregistry`。
- `pack()` 中把 wheel 和 rpm 拷进 server 打包目录，随 `AgentOS-Server-<arch>.tgz` 分发。

> rqlite rpm 分架构，注意 `ARCH` 用 `uname -m` 的 `x86_64`/`aarch64`（rpm 的 arch 标签正是这两个，无需像二进制那样映射成 amd64/arm64）。

### 2.2 配置 `deploy/agentos.sh`（注册模块）

`MODULES` 数组加入 `"agentregistry"`，位于 yuanrong 之后、jiuwenswarm 之前（gateway 运行期要写注册中心，故注册中心先起）：

```bash
MODULES=("jiuwenbox" "yuanrong" "agentregistry" "jiuwenswarm")
```

调度逻辑不动。

### 2.3 实现 `deploy/agentregistry/module.sh`（4 个钩子）

两个服务各由 **systemd** 托管：`rqlite.service`（存储）+ `a2x-registry.service`（注册中心）。模块目录随包携带两个静态文件：unit 模板 `a2x-registry.service` 与配置模板 `registry.env`（rqlite 用其 rpm 自带 unit）。注册中心 unit 声明 `After=rqlite.service` / `Requires=rqlite.service`，启动顺序交给 systemd。各钩子按需取用 agentos 全局量（`AGENTOS_ROOT`、`YR_PYTHON_VERSION`）与 `registry.env`，均不消费 `--hosts`；启停走 `systemctl`。

`a2x-registry.service` 关键字段：

```ini
[Unit]
After=rqlite.service
Requires=rqlite.service
[Service]
EnvironmentFile=/etc/a2x-registry/registry.env      # 配置唯一来源
ExecStart=/usr/bin/a2x-registry                      # 或 python3.11 -m a2x_registry.backend
Restart=on-failure                                   # 异常退出自动重启
RestartSec=3                                         # 重启前退避 3s
StartLimitIntervalSec=60                             # 熔断：60s 窗口内
StartLimitBurst=5                                    #   最多重启 5 次，超了停手（防疯狂重启）
[Install]
WantedBy=multi-user.target
```

> rqlite 的 rpm 自带 unit 通常已含 `Restart=on-failure`，如无则同样补上。

#### 2.3.1 `agentregistry_install`

- **输入**：`AGENTOS_ROOT`（whl / rpm 所在目录）、`YR_PYTHON_VERSION`。
- **流程**：① 从 `AGENTOS_ROOT` 装 rqlite rpm（`dnf install -y ./rqlite-*.rpm`，**系统级**）；② 系统级装注册中心 wheel（`pip install ./a2x_registry-*.whl`，使 `ExecStart` 路径稳定）；③ 拷 `a2x-registry.service` 到 `/etc/systemd/system/`、生成 `/etc/a2x-registry/registry.env`（已存在则不覆盖）；④ `systemctl daemon-reload`。
- **默认配置**：安装源恒为 `AGENTOS_ROOT` 本地文件（离线）；unit 装在 `/etc/systemd/system/`，配置在 `/etc/a2x-registry/registry.env`。

#### 2.3.2 `agentregistry_up`

- **输入**：`registry.env`（systemd `EnvironmentFile`，字段见下）。
- **流程**：① `systemctl enable --now a2x-registry`——因 `Requires=rqlite.service`，systemd **自动先起 rqlite** 再起注册中心，且 `enable` 使其开机自启；② curl 健康检查 `GET /api/datasets` 就绪后返回（`systemctl start` 不等 uvicorn 真就绪，故仍需探测）。
- **默认端口**（两个服务，均与其它模块**不冲突**）：

  | 服务 | 端口 | 默认绑定 | 备注 |
  |------|------|---------|------|
  | 注册中心 backend | **8000** | 本机网卡 IP | 对外，gateway / 用户访问 |
  | rqlite HTTP | **4001** | `127.0.0.1` | 仅本机 backend 连 |
  | rqlite Raft | **4002** | `127.0.0.1` | 单机自身；多节点用（后续） |

  > 现有占用：yuanrong `31182`/`8888`、jiuwenswarm `8888`。`8000`/`4001`/`4002` 均空闲。

- **默认配置**（`registry.env` 字段，backend 无 `--host`，全由此驱动）：

  | 变量 | 默认 | 说明 |
  |------|------|------|
  | `A2X_REGISTRY_BIND` | 本机网卡 IP | 禁 `0.0.0.0` |
  | `A2X_REGISTRY_PORT` | `8000` | backend 端口 |
  | `A2X_REGISTRY_MODE` | `appliance` | 建镜像/实例表 |
  | `A2X_REGISTRY_DB_KIND` | `rqlite` | 存储后端 |
  | `A2X_REGISTRY_DB_ENDPOINT` | `http://127.0.0.1:4001` | 连本机 rqlite |

#### 2.3.3 `agentregistry_down`

- **输入**：无。
- **流程**：`systemctl disable --now a2x-registry`（停并取消自启）；rqlite 视需要 `systemctl stop rqlite`。systemd 依赖关系保证注册中心先于 rqlite 停。
- **默认配置**：无。

#### 2.3.4 `agentregistry_uninstall`

- **输入**：`YR_PYTHON_VERSION`。
- **流程**：① `systemctl disable --now a2x-registry`；② 删 `/etc/systemd/system/a2x-registry.service` + `systemctl daemon-reload`；③ `pip uninstall -y a2x-registry`；④ `rpm -e rqlite`。
- **默认配置**：无。
