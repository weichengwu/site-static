# 静态网站发布工作流

在项目中添加 `.github/workflows/deploy.yml`，用 `uses` 引用本仓库的 [static-site.yml](.github/workflows/static-site.yml)。将 `REF` 换成包含这些参数的标签或提交 SHA，按项目的默认分支调整触发分支。

```yaml
name: Publish static site
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  site:
    uses: weichengwu/site-static/.github/workflows/static-site.yml@REF
    with:
      deploy-runner: site-deploy
      deploy-host: deploy.example.com
```

将示例中的 runner 标签和服务器地址换成项目的发布配置。能用 `build` 脚本生成 `dist/` 的 npm 或 pnpm 项目可以沿用其余默认值。共享工作流只在调用仓库的默认分支执行发布。

## 调用参数

在项目 YAML 的 `jobs.site.with` 下传参。实际发布时需要 `deploy-runner` 和 `deploy-host`。

| 参数 | 默认值 | 用途 |
| --- | --- | --- |
| `working-directory` | `.` | 从仓库根目录定位构建项目。lockfile 和 `package.json` 都从这里读取 |
| `output-dir` | `dist` | 构建产物目录，相对于 `working-directory` |
| `package-manager` | `auto` | 根据 lockfile 选择 npm 或 pnpm。两个 lockfile 同时存在时显式传 `npm` 或 `pnpm` |
| `node-version` | `24` | 构建所用 Node.js 版本 |
| `build-script` | `build` | `package.json` 里的构建脚本名称 |
| `build-env` | `{}` | 传给构建脚本的纯字符串环境变量 JSON 对象。用于站点 URL 等公开配置 |
| `required-files` | 空 | 除 `index.html` 外须存在且非空的产物文件，每行一个相对于 `output-dir` 的路径 |
| `site-name` | 仓库名 | 服务端使用的站点标识 |
| `deploy` | `true` | 设为 `false` 时只安装依赖、构建和校验 |
| `deploy-runner` | 空 | 执行发布的 runner 标签 |
| `deploy-host` | 空 | SSH 服务器的主机名或 IPv4 地址 |
| `ssh-user` | `ubuntu` | SSH 用户名 |
| `deploy-command` | `/usr/local/bin/deploy-static-site` | 服务器上的可执行文件路径，接收站点名和 ZIP 路径 |

npm 使用 `npm ci`，pnpm 使用 `pnpm install --frozen-lockfile`。pnpm 版本由项目的 `package.json` 声明。

### 自定义构建示例

- 构建输出在 `dist/client` 时，在 `with` 下加 `output-dir: dist/client`。
- 子目录中的站点可以参考下面的示例。调用仓库的 Actions Variables 中 `SITE_URL` 为空时，工作流只构建和校验。

```yaml
jobs:
  site:
    uses: weichengwu/site-static/.github/workflows/static-site.yml@REF
    with:
      deploy-runner: site-deploy
      deploy-host: deploy.example.com
      ssh-user: deploy
      working-directory: apps/website
      site-name: example-site
      deploy: ${{ vars.SITE_URL != '' }}
      build-env: ${{ vars.SITE_URL != '' && format('{{"SITE_URL":{0}}}', toJSON(vars.SITE_URL)) || '{}' }}
      required-files: |
        robots.txt
        sitemap.xml
```

## 发布环境

在发布 runner 上预先配置 SSH 私钥或 SSH agent，以及服务器的 `known_hosts`。调用方 YAML 里的地址和用户名会随项目代码可见，私钥保存在 runner 上。仅构建的运行使用 `ubuntu-latest`，可省略发布参数。发布时，安装依赖、构建打包、`scp` 和服务端部署在同一个 job 执行。ZIP 包含 `dist/index.html`，直接复制到服务器临时目录。工作流会清理本地和远端的临时包。

随附的 [deploy-static-site](server/deploy-static-site) 可安装为默认服务端命令。它依赖 Linux、Bash、GNU `mv`、`flock` 和 `unzip`。部署账号需能在 `/var/www` 下创建锁文件、临时目录以及替换目标站点。Web 服务器使用 `/var/www/<站点>/dist` 作为站点根目录。脚本先解包校验，再替换站点，切换失败时尝试恢复旧目录。切换期间可能短暂停顿。

自有部署程序可以通过 `deploy-command` 接入。程序需在执行期间使用传入的 ZIP，并通过退出码报告结果。
