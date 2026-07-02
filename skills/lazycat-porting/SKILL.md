---
name: lazycat-porting
description: >-
  懒猫微服 LPK 移植与上架流水线：Docker 转 manifest、本地 project build、lpk install 部署验证、
  appstore publish 提审。用户提到移植、本地打包、本地安装、上架审核、package.yml 元数据时使用。
  先读本 skill 三阶段流水线；inject/免密等审核硬性要求按需查 lazycat-library。
---

# 懒猫微服移植与上架（Porting）

## 定位

本 skill 覆盖**一条主链路**，不替代 59 篇完整库：

```
Docker/Compose → 写 YAML → 本地 build → 本地 install 验证 → 提审
```

| 阶段 | 本 skill | 按需查 library |
| --- | --- | --- |
| 移植配置 | Docker 映射、manifest 示例 | 路由细节、embed 镜像、数据库 |
| 本地打包部署 | build / install / log | lzc-build 字段、排障 FAQ |
| 提审 | publish、copy-image、资料清单 | **免密登录**、OIDC、审核指南全文 |

**不必**把 59 篇合成一份大文档；缺什么再按 `meta/index.json` 打开单篇。

## 三阶段流水线（主流程）

```
Task Progress:
- [ ] A. 移植：Docker → package.yml + lzc-manifest.yml + lzc-build.yml
- [ ] B. 本地验证：build → lpk install → 入口 / 持久化 / 日志
- [ ] C. 提审：开发者账号 → 镜像 copy-image（如需）→ 补全资料 → publish
```

### A. 移植（配置）

见下方「Docker 映射表」与 [porting-handbook.md §9](porting-handbook.md)。

最小目录：

```text
.
├── lzc-build.yml      # pkgout、icon、images（内嵌镜像时）
├── lzc-manifest.yml   # routes、services、ingress
├── package.yml        # package、version、name、description、locales
└── icon.png
```

### B. 本地打包与部署（必须先做）

```bash
# 1. 构建 LPK（发布包，非 dev deploy）
lzc-cli project build

# 2. 检查包内容
lzc-cli lpk info ./<package>-<version>.lpk

# 3. 安装到默认微服（本地验证，等同用户安装体验）
lzc-cli lpk install ./<package>-<version>.lpk

# 4. 确认运行
lzc-cli project info
lzc-cli project log -f
```

**通过标准**（提审前必达）：

- `lpk install` 成功，启动器能打开应用
- 重启应用后 `/lzcapp/var` 数据仍在
- 外部镜像在目标微服可拉取，或已内嵌到 LPK
- 加载时间合理（审核要求启动/响应不超过 5 分钟）

开发态 `project deploy` 仅用于迭代调试；**提审前必须用 `project build` + `lpk install` 走一遍发布包路径**。

### 移植验证实操（标准测试流程）

经 `qiaomu-app-review-insights` 跑通。**本地移植验证 = `build` + `install`**，不是 `project deploy`。

```bash
cd <项目根目录>

# 1. 构建发布包（读 lzc-build.yml，非 lzc-build.dev.yml）
lzc-cli project build
# → 产出 ./<package>-<version>.lpk

# 2. 安装到默认微服（等同用户安装体验）
lzc-cli lpk install ./<package>-<version>.lpk
# → 成功后会打印访问 URL，用微服账号登录

# 3. 可选：看日志
lzc-cli project log -f
```

**实测预期**（内嵌 Dockerfile 镜像时）：

| 步骤 | 耗时 | 说明 |
| --- | --- | --- |
| `project build` | 约 3～5 分钟 | 在目标微服远程构建镜像，打包进 LPK（~80MB 量级） |
| `lpk install` | 约 30～60 秒 | 安装正式包，输出 HTTPS 入口 |

**易混淆点**：

- `project info` 若存在 `lzc-build.dev.yml`，显示的是 **`.dev` 开发包**状态，与刚 `install` 的正式 LPK **不是同一个包**。
- 日常改代码 → `project deploy`；验证发布包 / 提审前 → **`project build` + `lpk install`**。

**自有 Dockerfile 移植最小配置**（内嵌镜像，无需 copy-image）：

```yaml
# lzc-build.yml
images:
  appreview:
    dockerfile: Dockerfile

# lzc-manifest.yml
services:
  appreview:
    image: embed:appreview
    binds:
      - /lzcapp/var/data:/lzcapp/var/data   # 左侧必须 /lzcapp 开头
```

### 最小手册阅读（够跑通验证，不必通读 59 篇）

| 目的 | 只读 |
| --- | --- |
| 开发 vs 发布分工 | `02-快速入门/04-开发流程.md` |
| LPK 构建/安装心智模型 | `02-快速入门/06-LPK 机制.md` |
| Dockerfile 内嵌镜像 | `02-快速入门/07-内嵌镜像进阶.md` |
| Docker → manifest 映射 | `03-发布应用/02-移植第一个应用.md` |
| 提审上架 | `03-发布应用/01-发布自己的第一个应用.md` + 审核指南 |

路由、数据库、免密登录等**遇到再查**，见下方「任务路由」。

### C. 创建应用与提审

**前置（人工，一次性）**：

1. 注册社区账号：https://lazycat.cloud/login
2. 开发者中心提交开发者申请：https://developer.lazycat.cloud/manage
3. 阅读审核指南：`01-欢迎/05-应用上架审核指南.md`

**package.yml 审核资料**（写入 LPK，非网页单独填）：

```yml
package: cloud.lazycat.app.example
version: 1.0.0
name: 示例应用
description: 一句话描述
locales:
  zh-CN:
    name: 示例应用
    description: 中文描述
  en-US:
    name: Example App
    description: English description
author: your-name
homepage: https://example.com
```

- `icon.png` 通过 `lzc-build.yml` 的 `icon` 字段打入 LPK（即 logo）
- 商店截图一般在开发者中心或 publish 流程补充，不在 LPK 内
- 名称、描述建议配 `locales`（BCP 47，如 `zh-CN` / `en-US`）

**外部镜像**（LPK 引用了 Docker Hub 等公网镜像时必做）：

```bash
lzc-cli appstore copy-image <公网可访问的镜像名>
# 输出 registry.lazycat.cloud/<用户名>/... 后，手动改 lzc-manifest.yml 里的 image
lzc-cli project build   # 改 manifest 后重新 build
```

**提交审核**（`lzc-cli 1.2.54+`）：

```bash
lzc-cli appstore publish ./<package>-<version>.lpk
```

**审核硬性要求**（速查未展开，提审前必查 library）：

| 要求 | 说明 | 查 |
| --- | --- | --- |
| 免密登录 | 上架必须支持，不能让用户手输密码 | `06-专题/01-免密登录.md` |
| 资料完整 | logo、名称、描述、截图 | 审核指南 §1 |
| 可安装可加载 | 依赖镜像可达、无白屏 | `09-常见问题/03-应用白屏.md` |
| 数据持久化 | 重启/升级不丢数据 | `05-最佳实践/02-文件访问.md` |

OIDC 或 inject 二选一实现免密；纯后端 API 类移植也要考虑 Web 登录体验。

提审逐项清单见 [publish-checklist.md](publish-checklist.md)。

## 手册根目录解析

按优先级确定 `{manual_root}`（Agent 读 index / 专栏 `.md` 时用）：

1. 工作区根目录存在 `meta/index.json` → `{manual_root}` = 工作区根
2. 环境变量 `LAZYCAT_MANUAL_ROOT` 已设置 → 使用该路径
3. 以上皆无 → 提示用户克隆本仓库或设置 `LAZYCAT_MANUAL_ROOT`，暂仅依赖 skill 内速查

索引路径：`{manual_root}/meta/index.json`

## 检索协议

1. **主链路问题**：读本文件三阶段 + [porting-handbook.md](porting-handbook.md)
2. **单点深入**：读 `meta/index.json`，打开 **1～2 篇** `.md`
3. **禁止**通读 59 篇

单次任务最多打开 **4 个文件**（含 index）。

## Docker 映射表

| Docker / Compose | LPK 配置 |
| --- | --- |
| `image` | `services.<name>.image` |
| `environment` / `--env` | `services.<name>.environment` |
| `volumes` / `--volume` | `services.<name>.binds`（左侧必须 `/lzcapp/...`） |
| HTTP/HTTPS `ports` / `--publish` | `application.routes` |
| 非 HTTP 端口（如 22） | `application.ingress` |
| 静态元数据 | `package.yml`（LPK v2） |
| 运行结构 | `lzc-manifest.yml` |
| 构建 / 图标 / 内嵌镜像 | `lzc-build.yml` |

**binds 规则**：左侧用 `/lzcapp/var` 或 `/lzcapp/cache`；右侧为容器内路径。

**routes 规则**：更具体路径写在更宽泛路径前（如 `/inspect/` 在 `/=` 之前）。

**service 互联**：应用内优先 `http://<service_name>:<port>`。

## 任务路由（按需）

| 场景 | 优先打开 |
| --- | --- |
| 发布 / copy-image / publish | `03-发布应用/01-发布自己的第一个应用.md` |
| 上架审核要求 | `01-欢迎/05-应用上架审核指南.md` |
| 免密登录（提审必做） | `06-专题/01-免密登录.md` |
| Docker 参数细节 | 同上 + [porting-handbook.md §9](porting-handbook.md) |
| 路由 / exec / http / file | `04-进阶主题/01-路由规则.md` 或 `02-快速入门/05-HTTP 路由.md` |
| MySQL / PostgreSQL / Redis | `05-最佳实践/03-数据库服务.md` |
| 文件与持久化边界 | `05-最佳实践/02-文件访问.md` |
| embed 镜像 / Dockerfile | `02-快速入门/07-内嵌镜像进阶.md` |
| manifest / package / build 字段 | `10-规范列表/03-lzc-manifest.yml.md` 等 |
| deploy-params 全量 | `10-规范列表/05-lzc-deploy-params.yml.md` |
| 4 层转发 / ingress | `04-进阶主题/07-4层转发.md` |
| 镜像拉取 / 内嵌 images | `10-规范列表/01-lzc-build.yml.md` |
| 开源移植参考 | https://gitee.com/lazycatcloud/appdb |

查法：先读 `meta/index.json`，按 `tags`/`task`/`summary` 匹配后只打开 `path` 指向的文件。

## 已知缺口（速查 vs 完整库）

| 主题 | 速查状态 | 补全来源 |
| --- | --- | --- |
| Redis 完整配置 | 已补全 | `05-最佳实践/03-数据库服务.md`（细节） |
| deploy-params 字段表 | 截断 | `10-规范列表/05-lzc-deploy-params.yml.md` |
| inject / 免密登录 | 提审硬性要求 | `06-专题/01-免密登录.md` |
| OIDC / 独立鉴权 | 未收录 | `04-进阶主题/06-对接 OIDC.md` |
| KVM / Dockerd | 未收录 | `07-传统模式/` |

本地 1～2 轮仍无法回答时，再查 https://developer.lazycat.cloud/ 并说明原因。

## 输出约定

帮用户移植时，优先给出：

1. 完整的 `package.yml` + `lzc-manifest.yml` 片段（可直接粘贴）
2. Docker → LPK 映射说明（哪条 volume 变成哪个 bind）
3. 验证命令清单
4. 若信息不足，明确列出还缺什么（端口协议、持久化路径、环境变量）

## 与本仓库其它 skill 的分工

| Skill | 用途 |
| --- | --- |
| **lazycat-porting**（本 skill） | Docker 移植三阶段流水线、build/install 验证、提审 |
| **lazycat-lpk-builder** | YAML 字段规范与打包护栏（references/） |
| **lazycat-library** | 59 篇开发者手册按需检索（需克隆手册仓库或设 `LAZYCAT_MANUAL_ROOT`） |
| **lazycat-lpk-netdisk** | 网盘 inject、COEP/ffmpeg.wasm 桥接 |

移植任务**优先读本 skill**；字段细节查 `lazycat-lpk-builder/references/`；深度专题查 `lazycat-library`。

## 安装

```bash
npx skills add zhistor26/lazycat-skills
```
