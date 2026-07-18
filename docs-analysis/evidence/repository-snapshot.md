# 仓库分析基线快照

本文件记录任务开始时的只读基线，用于区分“任务开始前已存在的状态”与“本任务新增的 `docs-analysis/` 文件”。

## 环境

- 工作目录：`/home/prome/agent3/OpenHands`
- 分析时间：2026-07-18（系统时钟）
- Git 用户：Mozi-Cloud

## Git 基线

- 分支：`main`
- HEAD commit：`2b31eb84eef6de2048c41d28033d9c3dc7444048`
- HEAD commit message：`docs: add multi-agent collaboration workflow feasibility analysis`
- 工作区状态（`git status --short`）：**clean**（无未提交修改）
- 子模块（`git submodule status --recursive`）：**无子模块**
- `.gitmodules`：不存在

## 仓库类型

- 单仓库（single repo，非 git submodule monorepo），但包含多个独立的 Python / Node 子项目。
- 跟踪文件总数：2562（`git ls-files | wc -l`）。

## 顶层目录文件分布（tracked）

| 目录 | 文件数 | 说明 |
| -- | -- | -- |
| `frontend/` | 1375 | React 19 + react-router 7 + Vite 前端 |
| `enterprise/` | 585 | 独立的 SaaS / 企业版 Python 服务（自带 poetry 项目） |
| `openhands/` | 277 | 主 Python 包（app_server / server / db / analytics） |
| `tests/` | 115 | 顶层测试 |
| `openhands-ui/` | 81 | 共享 UI 组件库 + design tokens |
| `.github/` | 36 | CI/CD 与配置 |
| `skills/` | 27 | OpenHands skills 定义 |
| `containers/`, `kind/`, `dev_config/`, `scripts/` | — | 容器 / k8s / 开发脚本 |

## 任务前已存在的 `docs-analysis/` 内容

- `docs-analysis/multi-agent-workflow-feasibility.md`（已提交，14KB，**任务前已存在，非本任务产物**）。

本任务不会修改或删除该文件；本任务仅向 `docs-analysis/` 新增指定的 11 个文档及 `evidence/`、`scripts/` 子目录内容。

## 写入边界确认

本任务所有写操作仅发生在：

- `docs-analysis/*.md`
- `docs-analysis/evidence/*`
- `docs-analysis/scripts/*`

不修改源码、配置、测试、CI、依赖锁文件、数据库、migration、子模块或 Git 配置；不执行 install / build / test / lint。
