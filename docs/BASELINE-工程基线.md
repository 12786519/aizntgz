# 工程基线

> 建立日期：2026-10-28 ｜ 状态：已建立

## 已完成

- 已建立根目录 AGENTS.md、docs、tasks、src、tests、scripts 和 .gitignore。
- 已锁定 Python 3.11 + FastAPI、React 18 + TypeScript、PostgreSQL 16 + pgvector、Redis 与 RabbitMQ。
- 已建立 Git 仓库。
- 已创建首次基线提交：ec1e94a（chore: establish project baseline）。

## 约束

- docs/requirements/需求文档.md 是只读业务输入。
- 现有 memory_bank/ 与 rag-saas-platform-plan/ 是资料库，业务实现不得依赖其临时展示文件。
- 工作区内嵌套的 aizntgz Git 仓库不属于本项目基线，不纳入本仓库提交。

## 复现检查

- Git 状态：git status --short
- 后续依赖安装：pip install -r requirements.txt
- 后续测试：pytest -q
- 后续静态检查：ruff check . && mypy src
