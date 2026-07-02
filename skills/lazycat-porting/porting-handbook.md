# 懒猫微服移植速查手册

本文根据已提供的懒猫微服开发者手册材料整理，定位为**移植向通用速查**。它不绑定具体业务实现，主要用于指导懒猫微服应用的开发、调试、打包、发布和移植。

说明：

- 原始材料来自网页拷贝，已去掉导航、分页、重复片段和明显网页噪音。
- 原始材料中 `数据库服务`、`Redis`、`deploy-params` 片段存在截断，本文只整理可确认内容。
- 命令、路径、配置字段名保持原文。
- **不全面**：inject、OIDC、多实例、KVM/Dockerd 等专题请通过 `lazycat-library` skill 查 `meta/index.json` 对应单篇。

## 1. 总体认知

懒猫微服应用面向用户侧通常表现为一个通过 HTTPS 访问的 Web App。应用最终交付格式是 `.lpk`，代码、静态资源、运行声明、镜像信息等会被打进 LPK 后安装到目标微服运行。

同一份 LPK 部署后，可在 Android、iOS、macOS、Windows、Web 浏览器等多端访问，具体平台支持也可以通过配置限制。

核心概念：

- `lzc-cli`：开发、构建、部署、发布、查看日志、进入容器的命令行工具。
- `lzc-build.yml`：构建配置，也是发布配置。
- `lzc-build.dev.yml`：开发态覆盖配置，`project deploy`、`project info`、`project exec` 等命令默认优先使用它。
- `lzc-manifest.yml`：应用运行结构配置，描述 `application`、`routes`、`services` 等。
- `package.yml`：应用静态元数据，LPK v2 新项目应把 `package`、`version`、`name`、`description` 等静态字段放在这里。
- `/lzcapp/var`：应用持久化数据目录，重启后保留。
- `/lzcapp/pkg/content`：LPK 打包内容在运行时的只读目录。

## 2. 开发环境搭建

目标：本地机器可以使用 `lzc-cli` 连接目标微服，并具备构建与部署能力。

前置条件：

- 有一台可访问的懒猫微服。
- 账号可以登录懒猫微服客户端。
- 终端可执行 `node` 和 `npm`。

步骤：

1. 安装 Node.js 18+，建议使用 LTS 版本。
2. 安装并登录懒猫微服客户端。
3. 在微服应用商店安装“懒猫开发者工具”。
4. 安装 `lzc-cli`：

```bash
npm install -g @lazycatcloud/lzc-cli
lzc-cli --version
```

如果 `lzc-cli --version` 能输出版本号，说明安装成功。

首次使用时准备 SSH Key：

```bash
[ -f ~/.ssh/id_ed25519.pub ] || ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
```

选择默认目标微服：

```bash
lzc-cli box list
lzc-cli box switch <boxname>
lzc-cli box default
```

如果在 WSL 或 LightOS 等无法直接使用 hclient API 的环境中开发，可用 SSH 方式添加目标微服：

```bash
lzc-cli box add-by-ssh root <微服局域网IP>
```

注意：`<微服局域网IP>` 必须填写微服局域网 IP，不要填写域名。该方式要求目标微服已开通 SSH。

仅当通过 hclient 模式接入目标微服时，才需要上传公钥：

```bash
lzc-cli box add-public-key
```

`add-by-ssh` 模式不需要、也无法成功执行 `lzc-cli box add-public-key`。

验证：

```bash
lzc-cli --version
lzc-cli box default
lzc-cli project --help
```

通过标准：

- `lzc-cli` 版本号可见。
- `box default` 能输出默认微服名。
- `project --help` 能看到 `deploy`、`start`、`info`、`exec`、`cp`、`log`、`release` 等命令。

常见问题：

- `No default box configured`：执行 `lzc-cli box list` 和 `lzc-cli box switch <boxname>`。
- `permission denied (publickey)`：hclient 模式执行 `lzc-cli box add-public-key`；`add-by-ssh` 模式检查目标微服 SSH 公钥授权配置。
- `lzc-cli: command not found`：确认 npm 全局安装路径已加入 `PATH`，或重新打开终端。

## 3. Hello World 与开发态验证

创建模板项目：

```bash
lzc-cli project create hello-lpk -t hello-vue
cd hello-lpk
```

模板默认会生成：

- `lzc-build.yml`：默认构建配置，也是发布配置。
- `lzc-build.dev.yml`：开发态覆盖配置，通常包含独立 dev 包名、空 `contentdir` 覆盖和 `DEV_MODE=1`。

部署并查看入口：

```bash
lzc-cli project deploy
lzc-cli project info
```

说明：

- 首次部署如果出现授权提示，按 CLI 输出打开浏览器完成授权。
- `project` 命令默认优先使用 `lzc-build.dev.yml`。
- 命令输出里的 `Build config` 表示本次实际命中的构建配置文件。
- 如需操作发布配置，显式加 `--release`。
- `project info` 在应用已运行时会输出 `Target URL`。

启动前端 dev server：

```bash
npm run dev
```

排查日志：

```bash
lzc-cli project log -f
```

查看 LPK 交付包：

```bash
lzc-cli project release -o hello.lpk
lzc-cli lpk info hello.lpk
```

## 4. HTTP 路由与后端对接

创建带后端模板：

```bash
lzc-cli project create hello-api -t todolist-golang
cd hello-api
```

模板默认路由示例：

```yml
application:
  routes:
    - /=exec://3000,/app/run.sh
```

`exec://3000,/app/run.sh` 的含义是执行 `/app/run.sh`，并把流量转发到 `127.0.0.1:3000`。

新增第三方 service 演示 `http` handler：

```yml
application:
  image: embed:app-runtime
  routes:
    - /inspect/=http://whoami:80/
    - /=exec://3000,/app/run.sh

services:
  whoami:
    image: registry.lazycat.cloud/traefik/whoami
```

说明：

- `/inspect/` 走 `http://whoami:80/`，不会触发 `/app/run.sh`。
- `http://` handler 只负责转发，不负责启动进程。
- `/inspect/` 要放在 `"/="` 之前，避免被更宽泛规则先匹配。

默认情况下，微服应用 HTTPS 路径受登录态保护。如需放开健康检查：

```yml
application:
  public_path:
    - /api/health
```

持久化目录：

```bash
lzc-cli project exec -s app /bin/sh
ls -la /lzcapp/var
```

后端数据文件应放在 `/lzcapp/var` 下。只有 `/lzcapp/var/` 目录下的内容会在重启应用后保留。

## 5. 路由规则

`application.routes` 字段为 `[]Rule` 类型，按照 `URL_PATH=UPSTREAM` 形式声明。

支持的上游协议：

- `file:///$dir_path`
- `exec://$port,$exec_file_path`
- `http(s)://$hostname/$path`

默认前缀裁剪规则：当浏览器请求 `/api/v1` 时，后端实际收到 `/v1`。如需保留前缀，使用 `application.upstreams` 并设置 `disable_trim_location: true`，要求 `lzcos v1.3.9+`。

### 5.1 http 上游

应用内 service 示例：

```yml
application:
  routes:
    - /=http://bitwarden.cloud.lazycat.app.bitwarden.lzcapp:80
    - /or_use_this_short_domain=http://bitwarden:80
  subdomain: bitwarden

services:
  bitwarden:
    image: bitwarden/nginx:1.44.1
```

### 5.2 file 上游

```yml
application:
  subdomain: pptist
  routes:
    - /=file:///lzcapp/pkg/content/
  file_handler:
    mime:
      - x-lzc-extension/pptist
    actions:
      open: /?file=%u
```

### 5.3 exec 上游

```yml
application:
  image: registry.lazycat.cloud/lzc/lzc-files:v0.1.47
  subdomain: file
  routes:
    - /api/=exec://3001,/lzcapp/pkg/content/backend
    - /files/=http://127.0.0.1:3001/files/
    - /=file:///lzcapp/pkg/content/dist
```

## 6. LPK 格式与文件边界

LPK v2 规则以前提 `lzcos v1.5.0+` 为基础。如需构建 LPK v2，应配合 `lzc-cli v2.0.0+`。

典型 LPK 顶层结构：

```text
.
├── manifest.yml
├── package.yml
├── content.tar | content.tar.gz
├── images/
├── images.lock
├── exports/
└── META/
```

LPK v2 中，静态字段应放在 `package.yml`，不应再放在 `manifest.yml`：

- `package`、`version`、`name`、`description`、`locales`、`author`、`license`、`homepage`、`min_os_version`、`unsupported_platforms`

## 7. 内嵌镜像与持久化边界

持久化边界：

- `/lzcapp/var` 会在重启后保留。
- 运行态目录，例如 `/home/lazycat`，手工改动会丢失，需要通过镜像或启动脚本固化。

文稿目录边界：

- `/lzcapp/documents` 是当前应用自己的应用文稿根目录，实际数据路径为 `/lzcapp/documents/<uid>`。
- 只有在 `package.yml.permissions` 中声明 `document.private` 后，系统才会提供 `/lzcapp/documents`。

## 8. 发布与提审（本地验证 → 上架）

### 8.1 本地打包与安装（提审前必做）

```bash
lzc-cli project build
lzc-cli lpk info ./<package>-<version>.lpk
lzc-cli lpk install ./<package>-<version>.lpk
lzc-cli project info
lzc-cli project log -f
```

说明：

- `project build` 产出**发布 LPK**；`project deploy` 是开发态，不能代替提审前验证。
- 本地 `lpk install` 模拟用户安装，务必确认能打开、数据能持久化、重启不丢。

### 8.2 开发者账号

1. 注册：https://lazycat.cloud/login
2. 开发者中心申请：https://developer.lazycat.cloud/manage

### 8.3 审核资料（package.yml + icon）

LPK 内必备：

- `package.yml`：`package`、`version`、`name`、`description`、`locales`（中英建议都配）
- `lzc-build.yml`：`icon: ./icon.png`
- 商店截图通常在 publish / 开发者中心流程补充，不在 LPK  tarball 里

`locales` 示例：

```yml
locales:
  zh-CN:
    name: 我的应用
    description: 中文描述
  en-US:
    name: My App
    description: English description
```

### 8.4 外部镜像

```bash
lzc-cli appstore copy-image <公网可访问镜像>
# 按输出修改 lzc-manifest.yml 中的 image，再 project build
```

### 8.5 提交审核

```bash
lzc-cli appstore publish ./<package>-<version>.lpk
```

提审前对照 `01-欢迎/05-应用上架审核指南.md`，重点：

- 资料完整（logo、名称、描述、截图）
- 可安装、可加载、无严重崩溃
- 数据持久化经重启验证
- **必须支持免密登录**（inject 或 OIDC，见 `06-专题/01-免密登录.md`）

## 9. 移植 Docker 应用

移植已有 Docker 应用时，重点把 Docker 参数转换为 `lzc-manifest.yml`。

Docker 参数对应关系：

| Docker | LPK |
| --- | --- |
| `--env` | `services.<name>.environment` |
| `--publish`（HTTP/HTTPS） | `application.routes` |
| `--publish`（非 HTTP，如 SSH 22） | `application.ingress` |
| `--volume` | `services.<name>.binds` |
| 镜像名 | `services.<name>.image` |

`binds` 左侧路径必须以 `/lzcapp` 开头，通常使用 `/lzcapp/var` 或 `/lzcapp/cache`。

GitLab service 示例：

```yml
services:
  gitlab:
    image: gitlab/gitlab-ee:17.2.8-patch7
    binds:
      - /lzcapp/var/config:/etc/gitlab
      - /lzcapp/var/logs:/var/log/gitlab
      - /lzcapp/var/data:/var/opt/gitlab
    environment:
      - GITLAB_OMNIBUS_CONFIG=external_url 'http://gitlab.example.com'
```

完整配置示例：

```yml
# package.yml
package: cloud.lazycat.app.gitlab
version: 17.2.8-patch1
name: GitLab
```

```yml
# lzc-manifest.yml
lzc-sdk-version: '0.1'
application:
  routes:
    - /=http://gitlab.cloud.lazycat.app.gitlab.lzcapp:80
  subdomain: gitlab
  ingress:
    - protocol: tcp
      port: 22
      service: gitlab

services:
  gitlab:
    image: gitlab/gitlab-ee:17.2.8
    binds:
      - /lzcapp/var/config:/etc/gitlab
      - /lzcapp/var/logs:/var/log/gitlab
      - /lzcapp/var/data:/var/opt/gitlab
    environment:
      - GITLAB_OMNIBUS_CONFIG=external_url 'http://gitlab.${LAZYCAT_BOX_DOMAIN}'; gitlab_rails['lfs_enabled'] = true;
```

目录结构：

```text
.
├── lzc-build.yml
├── lzc-manifest.yml
├── package.yml
└── icon.png
```

构建并安装：

```bash
lzc-cli project build
lzc-cli lpk install ./cloud.lazycat.app.gitlab-v17.2.8-patch1.lpk
```

从 `docker-compose.yml` 转换时，重点关注 `environment`、`ports`、`volumes`、`image`。

常见问题：

- 镜像拉不下来：自用场景可考虑 `lzc-build.yml` 的 `images` 内嵌；公用场景走官方 registry。
- 可参考官方移植开源仓库：`https://gitee.com/lazycatcloud/appdb`

## 10. 数据库服务（部分）

MySQL service 示例：

```yml
services:
  mysql:
    image: registry.lazycat.cloud/mysql
    binds:
      - /lzcapp/var/mysql:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=LAZYCAT
      - MYSQL_DATABASE=LAZYCAT
      - MYSQL_USER=LAZYCAT
      - MYSQL_PASSWORD=LAZYCAT
```

连接方式：`mysql.<package>.lzcapp:3306`

PostgreSQL 示例：

```yml
services:
  postgresql:
    image: registry.lazycat.cloud/postgres:18.1
    environment:
      - POSTGRES_USER=lazycat
      - POSTGRES_PASSWORD=lazycat
      - POSTGRES_DB=lazycat
    binds:
      - /lzcapp/var/pgdata:/var/lib/postgresql
```

Redis 示例：

```yml
services:
  redis:
    image: registry.lazycat.cloud/redis:8.0
    command: redis-server --appendonly yes
    binds:
      - /lzcapp/var/redis:/data
```

连接方式：`redis.<package>.lzcapp:6379`

## 11. 平台支持

```yml
unsupported_platforms:
  - ios
```

| 参数 | 平台 |
| --- | --- |
| `ios` | 不支持 iOS 和 iPad 移动端 |
| `android` | 不支持 Android 移动端 |
| `linux` | 不支持 Linux 桌面端 |
| `windows` | 不支持 Windows 桌面端 |
| `macos` | 不支持 macOS 桌面端 |
| `tvos` | 不支持懒猫智慧屏平台端 |

## 12. 部署参数（部分）

`lzc-deploy-params.yml` 用于定义安装时参数。完整字段见 `10-规范列表/05-lzc-deploy-params.yml.md`。

`DeployParam.type` 目前支持：`bool`、`lzc_uid`、`string`、`secret`。

## 13. 开发与发布检查清单

开发前：

- 已安装 Node.js 18+ 与 `@lazycatcloud/lzc-cli`。
- `lzc-cli box default` 指向目标微服。

移植时：

- Docker 参数已映射到 `services` / `routes` / `ingress` / `binds`。
- 持久化数据写入 `/lzcapp/var`。
- 更具体路由放在更宽泛路由前。

发布前：

- 静态元数据在 `package.yml`，运行结构在 `lzc-manifest.yml`。
- 执行 `lzc-cli project build` 与 `lzc-cli lpk info`。

## 14. 常用命令速查

```bash
npm install -g @lazycatcloud/lzc-cli
lzc-cli box list && lzc-cli box switch <boxname> && lzc-cli box default
lzc-cli project create hello-lpk -t hello-vue
lzc-cli project deploy && lzc-cli project info
lzc-cli project log -f
lzc-cli project build
lzc-cli lpk install ./your-app.lpk
lzc-cli appstore copy-image <公网镜像>
lzc-cli appstore publish ./your-app.lpk
```
