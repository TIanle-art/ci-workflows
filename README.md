# ci-workflows

各项目共用的 GitHub Actions 可复用 workflow。项目中只写调用，检查逻辑统一在这里维护。

## workflow 列表

- `baseline.yml`：所有项目都调用。用 gitleaks 扫描密钥（下载时校验 checksum），运行项目中的 `scripts/check-skills.sh` 检查公司 Skill 是否同步。
- `python-ci.yml`：FastAPI 等 Python 项目。`uv sync --locked`、Ruff、pyright、Alembic 迁移检查（只能有一个 head，并在测试数据库上执行 `upgrade head`）、pytest。默认启动 PostgreSQL 16 测试数据库。
- `node-ci.yml`：网页、小程序、桌面和 NestJS 项目。`pnpm install --frozen-lockfile`，有 Prisma 时生成客户端，再依次运行 `lint typecheck test build` 中存在的脚本。
- `main-guard.yml`：`main` 更新后检查最新提交是否来自已批准的 PR。免费计划的私有仓库没有分支保护，这是事后检查。
- `self-check.yml`：本仓库自己的检查，用 actionlint 检查 workflow 文件。

## 在项目中调用

handbook 仓库的 `project-kit/.github/workflows/` 中有现成的 `ci.yml` 和 `main-guard.yml`，复制到项目后按需删减。示例：

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  baseline:
    uses: TIanle-art/ci-workflows/.github/workflows/baseline.yml@v1

  backend:
    uses: TIanle-art/ci-workflows/.github/workflows/python-ci.yml@v1
    with:
      working-directory: services/api

  web:
    uses: TIanle-art/ci-workflows/.github/workflows/node-ci.yml@v1
    with:
      working-directory: apps/web
```

## 输入参数

`baseline.yml`

- `gitleaks-version`：gitleaks 版本，默认 `8.30.1`

`python-ci.yml`

- `working-directory`：Python 项目目录，默认 `.`
- `python-version`：默认 `3.12`
- `postgres`：是否启动测试数据库，默认 `true`

`node-ci.yml`

- `working-directory`：项目目录，默认 `.`
- `node-version`：默认 `22`
- `scripts`：依次运行的脚本，空格分隔，默认 `lint typecheck test build`，不存在的脚本自动跳过

`main-guard.yml` 没有参数，调用方需要给 `contents: read` 和 `pull-requests: read` 权限。

## 对项目的要求

Python 项目：

- 有 `pyproject.toml` 和 `uv.lock`，`ruff`、`pyright`、`pytest` 在开发依赖中。
- 测试读取环境变量 `DATABASE_URL`。CI 中的值是 `postgresql://app:app@localhost:5432/app`，项目需要的驱动前缀（例如 `postgresql+asyncpg://`）在配置代码中转换。
- 使用 Alembic 时，`alembic.ini` 放在 `working-directory` 中。

Node 项目：

- `package.json` 中有 `packageManager` 字段，例如 `"packageManager": "pnpm@10.x.x"`，CI 按它安装 pnpm。
- 仓库根目录有 `pnpm-workspace.yaml` 时，在根目录安装依赖，`pnpm-lock.yaml` 也在根目录；否则在 `working-directory` 中安装。
- 脚本名使用 `lint`、`typecheck`、`test`、`build`。

## 首次设置

1. 本仓库是私有仓库。在 Settings → Actions → General → Access 中，选择允许同一账号下其他仓库访问。个人账号下如果没有这个选项，就先把 workflow 文件复制到各项目，转到组织后再改为引用。
2. 合并第一版后，在 `main` 上打 `v1` 标签：

   ```bash
   git switch main && git pull
   git tag v1
   git push origin v1
   ```

## 版本规则

- 项目引用 `@v1`。
- 兼容的修改（修复、增加有默认值的参数）合并后，把 `v1` 标签移到新的提交：

  ```bash
  git tag -f v1 && git push -f origin v1
  ```

  标签由技术负责人操作，AI 不操作标签。

- 不兼容的修改（删除参数、改变默认行为）打 `v2`，在团队频道通知各项目迁移。

## 修改流程

1. 从 `main` 建分支，修改后提 PR。
2. PR 中 `self-check` 必须通过。
3. 先在一个测试项目中引用分支（`@<分支名>`）实际运行一次，再合并。
4. 由技术负责人审查。CI 改动会影响所有项目。
