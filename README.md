# ci-workflows

各项目共用的 GitHub Actions 可复用 workflow。项目中只写调用，检查逻辑统一在这里维护。

这是**公开仓库**，私有项目可以直接引用其中的可复用 workflow，不需要在 Settings → Actions → General → Access 里做任何设置。因为公开，这里不写客户信息、内网地址和密钥。

## workflow 列表

- `baseline.yml`：所有项目都调用。用 gitleaks 扫描密钥（下载时校验 checksum），运行项目中的 `scripts/check-skills.sh` 检查公司 Skill 是否同步。
- `python-ci.yml`：FastAPI 等 Python 项目。`uv sync --locked`、Ruff、pyright、Alembic 迁移检查（只能有一个 head，并在测试数据库上执行 `upgrade head`）、pytest；测试通过后，可以在仓库根目录再运行项目 Makefile 中的目标（`make-targets`，例如 `openapi-check`）。默认启动 PostgreSQL 17 测试数据库。
- `node-ci.yml`：网页、小程序、桌面和 NestJS 项目。`pnpm install --frozen-lockfile`，有 Prisma 时生成客户端，再依次运行 `lint typecheck test build` 中存在的脚本；设置了 `e2e-script` 时，再安装 Playwright 浏览器并运行端到端测试。`lint` 运行的是项目自己的脚本，用 Biome 还是 ESLint 由项目决定。
- `docker-smoke.yml`：构建镜像、启动容器、端到端冒烟测试。依次运行项目 Makefile 中的目标；失败时输出容器日志；最后总是停止并清理。
- `main-guard.yml`：`main` 更新后检查最新提交是否来自已批准的 PR；没有批准时，带豁免标签的 PR 放行并输出警告。免费计划的私有仓库没有分支保护，这是事后检查。
- `self-check.yml`：本仓库自己的检查，用 actionlint 检查 workflow 文件。

## 在项目中调用

handbook 仓库的 `project-kit/.github/workflows/` 中有现成的 `ci.yml` 和 `main-guard.yml`，复制到项目后按需删减。示例（FastAPI 项目，参考 template-backend-fastapi）：

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
      make-targets: "openapi-check"

  docker:
    uses: TIanle-art/ci-workflows/.github/workflows/docker-smoke.yml@v1
    with:
      targets: "caddy-check build docker-up smoke"
```

## 输入参数

`baseline.yml`

- `gitleaks-version`：gitleaks 版本，默认 `8.30.1`

`python-ci.yml`

- `working-directory`：Python 项目目录，默认 `.`
- `python-version`：默认 `3.14`
- `postgres`：是否启动测试数据库，默认 `true`
- `postgres-image`：测试数据库镜像，默认 `postgres:17`。应与生产环境的 PostgreSQL 主版本一致；启用 pgvector 时改为 `pgvector/pgvector` 的对应标签，例如 `pgvector/pgvector:pg17`
- `make-targets`：测试通过后，在**仓库根目录**依次运行的 make 目标，空格分隔，默认为空（不运行）。依赖已经按 `uv sync --locked` 安装好，环境变量与测试一致（只有 `DATABASE_URL` 和 `APP_ENV=test`）

`node-ci.yml`

- `working-directory`：项目目录，默认 `.`
- `node-version`：默认 `22`
- `scripts`：依次运行的脚本，空格分隔，默认 `lint typecheck test build`，不存在的脚本自动跳过
- `e2e-script`：端到端测试的脚本名，例如 `test:e2e`，默认为空（不运行）。不为空时，先运行 `pnpm exec playwright install --with-deps <浏览器>`，再运行 `pnpm run <e2e-script>`
- `playwright-browsers`：要安装的浏览器，空格分隔，默认 `chromium`

`docker-smoke.yml`

- `targets`：依次运行的 make 目标，空格分隔，默认 `build docker-up smoke`
- `logs-target`：任何一步失败时运行，用来输出容器日志，默认 `docker-logs`
- `down-target`：最后总是运行，用来停止并清理容器，默认 `docker-down`

`main-guard.yml`

- `waiver-label`：带有这个标签的 PR，没有批准也算通过（输出 `::warning::`），默认 `review-waived`；设为空字符串则不启用豁免。调用方需要给 `contents: read` 和 `pull-requests: read` 权限

## 对项目的要求

Python 项目：

- 有 `pyproject.toml` 和 `uv.lock`，`ruff`、`pyright`、`pytest` 在开发依赖中。
- 测试读取环境变量 `DATABASE_URL`。CI 中的值是 `postgresql://app:app@localhost:5432/app`，项目需要的驱动前缀（例如 `postgresql+asyncpg://`）在配置代码中转换。
- CI 只提供 `DATABASE_URL` 和 `APP_ENV=test` 两个环境变量，**只有这两个变量时，Alembic 迁移检查和全部测试也要通过**。所以 Alembic 只能依赖数据库配置，测试需要的其他配置（例如密钥）在 `conftest.py` 里设置测试值，不能为此放宽生产环境的配置校验。参考 template-backend-fastapi 把配置拆成 `DatabaseSettings` 和 `Settings` 的做法。
- 使用 Alembic 时，`alembic.ini` 放在 `working-directory` 中。
- 用到 `make-targets` 时，目标要在仓库根目录的 Makefile 里。

Node 项目：

- `package.json` 中有 `packageManager` 字段，例如 `"packageManager": "pnpm@10.x.x"`，CI 按它安装 pnpm。
- 仓库根目录有 `pnpm-workspace.yaml` 时，在根目录安装依赖，`pnpm-lock.yaml` 也在根目录；否则在 `working-directory` 中安装。
- 脚本名使用 `lint`、`typecheck`、`test`、`build`；端到端测试的脚本名通过 `e2e-script` 指定。

使用 `docker-smoke.yml` 的项目：

- 仓库根目录的 Makefile 提供 `build`、`docker-up`、`smoke`、`docker-logs`、`docker-down`（可以在 `targets` 等参数里改名）。目标名称和 template-backend-fastapi 一致，`docker-up` 要先执行数据库迁移，`smoke` 做端到端检查。

## main 守卫和豁免标签

只有一名维护者时，作者不能批准自己的 PR，守卫必然失败；一直红着会让人习惯性忽略它。所以带豁免标签（默认 `review-waived`）的 PR 没有批准也通过，但会输出警告，提醒技术负责人确认原因。

- 谁可以加标签、什么情况可以加，规则写在《团队协作开发规范》。本 workflow **不检查是谁加的标签**，只读取标签。
- 守卫在 `main` 更新时读取 PR 当前的标签。合并之后才加标签的，加完后在 Actions 页面重新运行这次守卫即可。
- 没有批准、也没有豁免标签时仍然失败，错误信息里会提示可以加豁免标签。

## 首次设置

`v1` 标签已经发布，项目直接引用 `@v1`。

## 版本规则

- 项目引用 `@v1`。
- 兼容的修改（修复、增加有默认值的参数）合并后，把 `v1` 标签移到新的提交：

  ```bash
  git tag -f v1 && git push -f origin v1
  ```

  标签由技术负责人操作，AI 不操作标签。

- 不兼容的修改（删除参数、改变默认行为）打 `v2`，在团队频道通知各项目迁移。
- **v1.1 的例外：** `python-ci.yml` 的 `python-version` 默认值从 3.12 改为 3.14，测试数据库默认值从 `postgres:16` 改为 `postgres:17`。严格说这是改变默认行为，但当时还没有项目在使用 v1，所以仍然放在 v1 上。以后改变默认行为的修改，打 v2。

## 修改流程

1. 从 `main` 建分支，修改后提 PR。
2. PR 中 `self-check` 必须通过。
3. 先在一个测试项目中引用分支（`@<分支名>`）实际运行一次，再合并。
4. 由技术负责人审查。CI 改动会影响所有项目。

## 待办

- 依赖漏洞扫描（规范 6.7 的 CI 检查表里有，现在没有）。
- 构建并推送镜像的 workflow（规范 13.4）。目前由维护者在本机构建并推送。
- 客户端生成代码是否最新的检查（`@hey-api/openapi-ts`），在做网页模板时加入 `node-ci.yml` 或单独的 workflow。
