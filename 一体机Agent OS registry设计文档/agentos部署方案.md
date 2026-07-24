# agentos 部署注册中心方案

> 说明 A2X 注册中心如何接入一体机 **agentos** 统一部署框架。非注册中心组件（yuanrong / jiuwenswarm / jiuwenbox）从略。

## 1. agentos 整体工作流程

agentos 分**构建**和**部署**两个阶段：一个造包，一个用包。

```
build.sh（构建机，联网）                       agentos.sh（目标机）
  下载各组件 wheel ──打包──▶ AgentOS-Server.tgz ──解压──▶ install → up
```

### 构建阶段（`build/build.sh`，在联网的构建机上跑）

- **第一步**：解析参数、按模式（`daily` / `release`）确定各组件的下载地址与文件清单。**【需要修改】** 把注册中心 wheel 的下载地址（`REGISTRY_WHL_URL`）加入清单。
- **第二步**：从 OBS / gitcode 下载各组件的 wheel（openyuanrong、jiuwenswarm 等）。**【需要修改】** 增加下载注册中心 wheel。
- **第三步**：把下载的 wheel 连同仓库的 `deploy/` 目录，打包成 `AgentOS-Server-<arch>.tgz`。**【需要修改】** 打包时纳入注册中心 wheel。

> 这样注册中心和其它组件一致：构建时下载、打进包，部署时从本地 `AGENTOS_ROOT` 安装。

### 部署阶段（`deploy/agentos.sh`，在目标机上跑）

- **第一步**：把 `AgentOS-Server-<arch>.tgz` 传到目标机并解压，得到 wheel 包和 `deploy/` 目录。
- **第二步**：`bash agentos.sh install` —— 按 `MODULES` 声明顺序，逐个模块本机 pip 安装各自的 wheel（只装包，不起服务）。**【需要修改】** 新增 agentregistry 模块的 `install` 钩子（从 `AGENTOS_ROOT` 本机安装注册中心 whl）。
- **第三步**：`bash agentos.sh up` —— 按 `MODULES` 声明顺序，逐个模块启动服务。**【需要修改】** 新增 agentregistry 模块的 `up` 钩子（后台起注册中心 backend）。
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

| 钩子 | 行为 |
|------|------|
| `agentregistry_install` | 从 `AGENTOS_ROOT` 安装：① rqlite rpm（`rpm -i` / `dnf install ./rqlite-*.rpm`）；② 注册中心 wheel（`pip install a2x_registry-*.whl`）|
| `agentregistry_up` | ① 起 rqlite（`rqlited`，指定数据目录、HTTP `:4001`）；② 起注册中心 backend（`python -m a2x_registry.backend`，env 指向本机 rqlite）；③ 健康检查就绪后返回 |
| `agentregistry_down` | 停注册中心 backend + 停 rqlite |
| `agentregistry_uninstall` | `pip uninstall` 注册中心 + `rpm -e` rqlite |

配置（环境变量驱动，backend 无 `--host`）：

| 变量 | 含义 | 默认 |
|------|------|------|
| `A2X_REGISTRY_BIND` | 注册中心监听地址（禁 `0.0.0.0`） | 本机网卡 IP |
| `A2X_REGISTRY_PORT` | 注册中心监听端口 | `8000` |
| `A2X_REGISTRY_MODE` | `appliance`（建镜像/实例表） | `appliance` |
| `A2X_REGISTRY_DB_KIND` | 存储后端 | `rqlite` |
| `A2X_REGISTRY_DB_ENDPOINT` | rqlite HTTP 地址 | `http://127.0.0.1:4001` |
