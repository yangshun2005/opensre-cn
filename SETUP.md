# 开发环境搭建

## 前置条件

- **Python 3.12+** — [`pyproject.toml`](pyproject.toml) 要求（`requires-python = ">=3.12"`）。CI 工作流使用 **Python 3.13**（见 [`.github/workflows/ci.yml`](.github/workflows/ci.yml)）。[`.tool-versions`](.tool-versions) 固定了 Python **3.13**、**uv**、**ruff** 和 **mypy**（版本尽量与 [`uv.lock`](uv.lock) 对齐），以及供 mise/asdf 类管理器使用的 Node/pnpm——可选；常规流程通过 **`make install`** / **`uv sync`** 将 **ruff** 和 **mypy** 安装到 `.venv`。
- Git
- **[uv](https://docs.astral.sh/uv/getting-started/installation/)** — `make install` 必需（从 `uv.lock` 安装锁定依赖）
- **Make** — macOS/Linux 自带；Windows 见下方选项

## 支持的平台和架构

发布可用性与默认 CI 覆盖范围是两回事。请按下方对应平台的安装路径操作，然后按匹配的安装指南完成首次运行流程。[快速开始](docs/getting-started/quickstart.mdx) 有相同流程的截图。

用 curl 安装已发布的 CLI，然后运行 `opensre`：

```bash
curl -fsSL https://install.opensre.com | bash
```

```bash
opensre
```

| 操作系统 | 架构 | 安装指南 | 说明 |
| --- | --- | --- | --- |
| macOS | arm64, x86_64 | [macOS](https://www.opensre.com/docs/install/macos) | 安装器会选择匹配的二进制文件。 |
| Linux | x86_64, arm64 | [Linux](https://www.opensre.com/docs/install/linux-local) | 需要 glibc 2.35+（Ubuntu 22.04+ 或同等版本）；不支持 Alpine/musl。 |
| Windows | x86_64 | [Windows](https://www.opensre.com/docs/install/windows-local) | 在 PowerShell 中：`irm https://install.opensre.com/install.ps1 \| iex` |

curl 安装器默认使用滚动的 `main` 构建。以下步骤面向从源码检出工作的贡献者。

主 CI 主要运行在 `ubuntu-24.04`。Windows CI 是可选的，仅在 PR 带有 `ci:windows` 标签时运行，因此它是有用的信号，而非默认覆盖所有平台的保证。

## 快速搭建（所有平台）

1. Fork 并克隆：

```bash
git clone https://github.com/YOUR_USERNAME/opensre.git
cd opensre
```

2. 如需要，安装 uv：

- **macOS/Linux：** `curl -LsSf https://astral.sh/uv/install.sh | sh`（或参考 [uv 安装指南](https://docs.astral.sh/uv/getting-started/installation/)）
- **Windows（PowerShell）：** `irm https://astral.sh/uv/install.ps1 | iex`  
  或：`winget install --id astral-sh.uv -e`

3. 安装依赖：

```bash
make install
```

没有 Make 时（等价于 `make install`）：

```bash
uv sync --frozen --extra dev
uv run python -m infrastructure.analytics.install
```

4. 验证：

```bash
make lint && make format-check && make typecheck && make test-cov
```

`format-check` 是 CI 强制执行的格式检查；提交 PR 前请先运行。

---

## VS Code 开发容器

1. 在 VS Code 中安装 **Dev Containers** 扩展。
2. 在宿主机上启动 Docker Desktop、OrbStack、Colima 或其他兼容 Docker 的运行时。
3. 打开仓库并运行 **Dev Containers: Reopen in Container**。

镜像基于 [`.devcontainer/Dockerfile`](.devcontainer/Dockerfile) 构建（**Python 3.13**）。**`postCreateCommand`** 会创建 `.venv-devcontainer` 并运行 **`pip install -e '.[dev]'`**（不使用 `uv`）。VS Code 使用的解释器是 `.venv-devcontainer/bin/python`。

在宿主机上，大多数贡献者使用 **`make install`** + **`uv run`**；两种方式都有效。

---

## Windows 专用设置

Windows 不自带 **make**。请选择以下一种方式。

### 选项 A：Chocolatey（推荐）

1. **以管理员身份**打开 PowerShell。
2. 安装 Chocolatey（请先审阅脚本）：

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

3. 安装 make：

```powershell
choco install make
```

4. 重启终端并验证：`make --version`。

### 选项 B：winget

```powershell
winget install GnuWin32.Make
```

重启终端，然后运行 `make --version`。

### 选项 C：不使用 Make

在仓库根目录运行等价命令（`uv` 需在同一 shell 的 `PATH` 中）。尽量优先使用 **`make test-cov`**——完整的 pytest 命令在 [`Makefile`](Makefile) 的 `test-cov` 目标中（`pytest -n auto`、覆盖率及忽略项）。

```bash
uv sync --frozen --extra dev
uv run python -m infrastructure.analytics.install

uv run ruff check config core gateway integrations infrastructure surfaces tools tests/
uv run ruff format --check config core gateway integrations infrastructure surfaces tools tests/
uv run mypy config core gateway integrations infrastructure surfaces tools

uv run pytest -n auto -v \
  --cov=config --cov=core --cov=gateway --cov=integrations \
  --cov=infrastructure --cov=surfaces --cov=tools --cov-report=term-missing
```

---

## 故障排查

### 命令未使用项目环境

- 在仓库根目录优先使用 **`uv run <command>`**。
- 刷新依赖：**`uv sync --frozen --extra dev`**。

### Command not found: python

- 安装 Python **3.12+** 并确保其在 `PATH` 中（`python --version`）。

### Command not found: uv

- 安装 uv（链接见上），然后重启终端。

### `make install` / `uv sync` 失败

- 在仓库根目录运行命令；确保 **`uv.lock`** 存在。
- 升级 uv：**`uv self update`**。
- 如果锁文件与 **`pyproject.toml`** 不匹配，在本地运行 **`uv lock`** 并提交更新后的锁文件（或提交 PR）。

### make: command not found（Windows）

- 安装 make（见上）或使用选项 C。

### 运行代码时出现导入错误

- 在仓库根目录使用 **`uv run`**。
- 重新运行 **`uv sync --frozen --extra dev`**。

### 启动能力警告（`curl`、`shell`、`network`、`python` 缺失）

当 `PATH` 中的工具缺失或沙箱探测失败时，启动过程会记录非致命警告。沙箱相关行如下：`<capability> is unavailable in this environment (probe returned unavailable) — the agent will not be able to use it.` 在正常机器上，两条 network 行是**预期**的。

| 启动警告 | 原因 | 影响 | 处理方式 |
| :--- | :--- | :--- | :--- |
| **`curl is not on PATH`** | `curl` 不在 `PATH` 中。 | 会告知 agent 不要通过 shell 调用 `curl`。 | **macOS：** 使用内置的 `/usr/bin/curl`<br />**Linux：** `sudo apt-get install -y curl`<br />**Windows：** `winget install cURL.cURL` |
| **`no interactive shell (bash/sh) on PATH`** | `bash` 和 `sh` 都不在 `PATH` 中。 | 会告知 agent 无法运行 shell 命令。 | **Linux：** `sudo apt-get install -y bash`<br />**macOS：** 保持 `/bin` 在 `PATH` 中。<br />**Windows：** Git Bash（`winget install Git.Git`）或 WSL。 |
| **`network egress is blocked for sandboxed code by default`** | 默认沙箱策略阻止出站套接字。 | 沙箱中的 Python 无法打开原始套接字。 | 预期行为，忽略即可。出站 HTTP 请使用已配置的集成。**不要**设置 `OPENSRE_ALLOW_NETWORK=1`——那只会隐藏警告。 |
| **`network requests is unavailable in this environment`** | 沙箱网络探测使用相同的默认阻止。 | 同上。 | 预期行为，同上。 |
| **`python execution is unavailable in this environment`** | 沙箱无法运行短 Python 片段，通常是因为 OpenSRE 临时目录不可写。该目录是 Python 进程临时目录（`tempfile.gettempdir()`）下的 `opensre`，可能是 `$TMPDIR`、`%TEMP%`、`%TMP%` 或 `/tmp`——不是固定路径。 | 会告知 agent 无法运行沙箱 Python。 | 创建进程实际使用的目录（会打印路径）：`uv run python -c "from pathlib import Path; import tempfile; p = Path(tempfile.gettempdir()) / 'opensre'; p.mkdir(parents=True, exist_ok=True); p.chmod(0o700); print(p)"`<br />然后运行 `make install`（或 `uv sync --frozen --extra dev`）。 |
| **`shell commands is unavailable in this environment`** | `PATH` 中没有 `bash`/`sh`（与 shell 行相同的检查）。 | 会告知 agent 无法运行 shell 命令。 | 安装步骤同 **`no interactive shell (bash/sh) on PATH`**。 |
| **`file reading is unavailable in this environment`** | 进程无法列出当前工作目录。 | 会告知 agent 无法读取本地文件。 | 在可读的检出目录中运行（`ls .` 应成功）。 |

### `opensre` 未使用本地代码修改

`make install` 以**可编辑**模式将本仓库安装到 `.venv`，但 **`PATH`** 中可能更早出现另一个 **`opensre`**（安装器二进制、版本管理器、`~/.local/bin` 等）。

1. 在仓库根目录优先使用 **`uv run opensre …`**。
2. 或将 venv 前置：`export PATH="$(pwd)/.venv/bin:$PATH"`（macOS/Linux），然后 **`hash -r`** / 新开 shell，并确认 **`which opensre`** 指向 **`<repo>/.venv/bin/opensre`**。

---

## 验证环境

```bash
make lint && make format-check && make typecheck && make test-cov
```

如果这些通过，就可以开始开发了。贡献流程：**[CONTRIBUTING.md](CONTRIBUTING.md)**。更深入的贡献者主题（基准测试、部署、遥测细节）：**[docs/DEVELOPMENT.md](docs/DEVELOPMENT.md)**。

---