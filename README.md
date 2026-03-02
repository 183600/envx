# envx

<p align="center">
  <strong>告别开发环境混乱，一键切换项目上下文</strong>
</p>

`envx` 是一个通过配置文件自动化开发环境上下文切换的CLI工具。它专为多项目并行开发设计，解决了环境变量冲突、后台服务残留和命令历史混杂的痛点。通过 Shell 钩子与配置驱动，让每个项目都拥有独立、纯净的运行环境。

---

## ✨ 核心功能

-   **🔧 配置化变量注入**: 解析 `dev-env.yaml`，动态加载环境变量，支持作用域隔离。
-   **♻️ 进程生命周期管理**: 自动清理旧环境依赖进程，启动新服务，解决端口占用与服务残留问题。
-   **📂 目录与历史隔离**: 自动切换工作目录，并将 Shell 历史记录重定向至项目独立文件，避免历史命令污染。
-   **🚀 交互式初始化**: 提供 `init` 命令快速生成配置模板，支持常见服务预设。

---

## 📦 安装

### 通过 Homebrew (macOS/Linux)
```bash
brew tap your-org/tap
brew install envx
```

### 通过 Go 安装
```bash
go install github.com/your-org/envx@latest
```

### 手动下载
从 [Releases](https://github.com/your-org/envx/releases) 页面下载对应平台的二进制文件，并将其放入 `PATH` 中。

---

## 🚀 快速开始

### 1. 初始化配置
在你的项目根目录运行：

```bash
envx init
```

这将启动一个交互式向导，帮助你生成 `dev-env.yaml` 文件。

### 2. 配置 Shell 集成 (关键步骤)
为了使 `envx` 能够修改当前 Shell 的工作目录和环境变量，你需要将以下内容添加到你的 Shell 配置文件（如 `~/.zshrc` 或 `~/.bashrc`）：

```bash
# 定义切换函数
function switch() {
    eval "$(envx switch "$@")"
}
```

### 3. 切换环境
在项目目录下执行：

```bash
switch my-project
```

工具将自动：
1. 清理上一个项目的后台进程。
2. 启动当前项目依赖的服务。
3. 注入环境变量。
4. 切换目录并隔离历史记录。

---

## 📝 配置文件说明 (`dev-env.yaml`)

`envx` 的核心是配置文件。以下是一个典型的配置示例：

```yaml
# 项目名称
project: "frontend-react"

# 1. 目录与历史隔离
root_path: "~/projects/frontend-react"  # 自动 cd 到此目录
history_file: ".history_dev"            # 历史记录保存路径（相对于 root_path）

# 2. 环境变量注入
env_vars:
  NODE_ENV: "development"
  API_ENDPOINT: "http://localhost:3000"
  REACT_APP_VERSION: "1.0.0"

# 3. 依赖进程生命周期管理
procs:
  - name: "node-server"        # 进程别名，用于日志标识
    command: "npm run start"   # 启动命令
    cwd: "./server"            # 工作目录（相对于 root_path）
    
  - name: "redis-docker"
    command: "docker run -p 6379:6379 redis:alpine"
    cwd: "."
```

---

## 💡 使用场景示例

### 场景：前后端并行开发

你正在同时维护一个 **React 前端** 和一个 **Go 后端**。

#### 1. 进入前端项目
当你结束后端工作，准备开发前端时：

```bash
switch frontend
```

**`envx` 执行流程：**
1.  **清理**: 查找并停止之前 `envx` 启动的 PostgreSQL 进程（Go 后端依赖）。
2.  **启动**: 执行 `npm run dev`，启动前端开发服务器。
3.  **变量**: 设置 `NODE_ENV=development`，`PORT=3000`。
4.  **历史**: 将 `HISTFILE` 设置为 `~/projects/frontend/.history_dev`。
5.  **目录**: 自动 `cd` 至 `~/projects/frontend`。

#### 2. 切换至后端项目
现在你需要切换回后端修复 Bug：

```bash
switch backend
```

**`envx` 执行流程：**
1.  **清理**: 终止 Node.js 进程（前端依赖），释放端口。
2.  **启动**: 启动 PostgreSQL Docker 容器（后端依赖）。
3.  **变量**: 加载 Go 相关路径、数据库连接字符串等变量。
4.  **隔离**: 切换目录并重置历史记录文件至后端项目。

---

## 🛠 命令详情

### `envx init`
交互式生成 `dev-env.yaml` 模板。
- 支持选择预设模板（Web, Database, Microservice 等）。
- 自动检测当前目录结构推荐配置。

### `envx switch [name]`
核心切换命令。通常通过 Shell 函数 `eval` 调用。
- 输出 Shell 脚本代码供 `eval` 执行。
- 管理后台进程的 PID 文件（存储于 `/tmp/envx-pids/`）。
- 检查配置文件热更新。

### `envx status`
查看当前由 `envx` 管理的进程状态和环境变量列表。

### `envx stop`
手动停止当前项目环境上下文关联的所有后台进程。

---

## 🧩 工作原理

`envx` 采用了 "Wrapper Script" 模式来解决 CLI 无法直接修改父 Shell 状态的限制：

1.  **配置解析**: `envx` 读取 `dev-env.yaml`。
2.  **进程管理**: 
    - 检查 PID 锁文件，向旧进程发送 `SIGTERM` 信号。
    - 使用 `exec` 启动新进程，并记录新 PID。
3.  **脚本生成**: `envx switch` 并不直接执行 `export` 或 `cd`，而是生成一段标准的 Shell 脚本输出到 stdout。
4.  **Shell 注入**: 
    ```bash
    eval "$(envx switch)"
    ```
    这段代码捕获 `envx` 的输出并在当前 Shell 会话中立即执行，从而实现环境变量注入和目录切换。

---

## 📄 License

MIT License © 2023 Your Org
