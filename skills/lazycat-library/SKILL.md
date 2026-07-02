---
name: lazycat-library
description: >-
  查询懒猫微服开发者手册（LPK 开发、移植、上架、路由、inject、KVM/Dockerd、规范字段）。
  用户提到 lzc-cli、LPK、lzc-manifest、lzc-build、package.yml、移植、路由、免密登录、
  应用商店、懒猫微服开发时使用。必须先读本地 meta/index.json 再按需打开单篇，禁止通读全库；
  Docker 移植工作流优先用 lazycat-porting skill；本地解决不了再查 https://developer.lazycat.cloud/。
---

# 懒猫微服开发者手册（Library）

## 手册根目录解析

按优先级确定 `{manual_root}`：

1. 工作区根目录存在 `meta/index.json` → `{manual_root}` = 工作区根
2. 工作区根目录存在 `lazycat-developer-manual/meta/index.json` → `{manual_root}` = 工作区根下的 `lazycat-developer-manual/`
3. 环境变量 `LAZYCAT_MANUAL_ROOT` 已设置 → 使用该路径
4. 常见克隆路径（若存在）：`D:/懒猫微服移植仓库/lazycat-developer-manual/`
5. 以上皆无 → 告知用户需克隆手册仓库或设置 `LAZYCAT_MANUAL_ROOT`

索引路径：`{manual_root}/meta/index.json`

## 与 lazycat-porting 的分工

| Skill | 用途 |
| --- | --- |
| **lazycat-porting** | Docker/Compose → LPK 移植工作流、三阶段流水线、提审 checklist |
| **lazycat-lpk-builder** | 自包含 YAML 规范（`references/`），不依赖手册仓库 |
| **lazycat-library**（本 skill） | 59 篇完整手册按需检索；规范字段、inject、排障等 |

用户明确要「移植 docker 应用」时，**先** `lazycat-porting`，再按需回落本 skill 单篇；仅查字段定义时可先读 `lazycat-lpk-builder`。

## 官方开发者网站（兜底来源）

- 地址：https://developer.lazycat.cloud/
- **绝大多数内容本地已覆盖**；仅在本地查不到、信息过时、或反复尝试仍无法解决时，才去官网核对

## 信息来源优先级（必须遵守）

1. **移植任务**：`lazycat-porting` → `porting-handbook.md` → 本库单篇
2. **本地手册**（首选）：`meta/index.json` → 匹配到的 `.md` 单篇
3. **本地缺口**：`index.json` 无匹配、或文章在已知缺口列表 → 告知用户
4. **官方网站**（兜底）：本地 1～2 轮检索仍无法回答时
5. **禁止**：跳过本地直接上网查；禁止为省事通读官网全站

## 检索协议（必须遵守）

1. **第一步**：读取 `meta/index.json`（约 55KB，含 59 篇文章摘要）
2. **第二步**：按 `tags`、`task`、`summary` 匹配，选出 **1～3 篇教程**；若涉及 YAML 字段，再加 **1 篇** `10-规范列表/`
3. **第三步**：只读取 `path` 指向的 `.md` 文件正文
4. **禁止**：一次性读取全部专栏或递归读整个目录

单次任务建议最多打开 **4 个文件**（含 index）。

## 专栏编号

| 编号 | 专栏 | 用途 |
|------|------|------|
| 01 | 欢迎 | 理念、上架审核、激励 |
| 02 | 快速入门 | 第一次开发 LPK（必读路线） |
| 03 | 发布应用 | 发布、移植 Docker/Compose |
| 04 | 进阶主题 | 路由、鉴权、inject、多实例等 |
| 05 | 最佳实践 | 文件访问、数据库、排障 |
| 06 | 专题 | 免密登录、文件选择器、Skill/MCP |
| 07 | 传统模式 | SSH、KVM、Dockerd |
| 08 | 懒猫生态 | 客户端能力、官方扩展 |
| 09 | 常见问题 | FAQ、网络、白屏、启动脚本 |
| 10 | 规范列表 | lzc-build / manifest / package 等字段定义 |

文章文件名格式：`{序号}-{标题}.md`，如 `02-快速入门/05-HTTP 路由.md`。

## 任务路由

| 用户意图 | 优先专栏 / 文章 |
|----------|-----------------|
| 移植 Docker/Compose | **先** `lazycat-porting` → `03-发布应用/02-移植第一个应用.md` |
| 第一次做 LPK | `02-快速入门/01-路线总览.md` → 按路线顺序 |
| 开发调试 | `02-快速入门/04-开发流程.md` |
| 本地 build / install / 提审 | **先** `lazycat-porting` → [publish-checklist.md](../lazycat-porting/publish-checklist.md) |
| HTTP 路由 / 后端 | `02-快速入门/05-HTTP 路由.md` 或 `04-进阶主题/01-路由规则.md` |
| inject / 免密 | `06-专题/01-免密登录.md`、`04-进阶主题/14-脚本注入.md` |
| YAML 字段含义 | `10-规范列表/` 对应文件 |
| KVM / Dockerd 折腾 | `07-传统模式/` |
| 客户端 JS SDK | `08-懒猫生态/02-应用接入客户端能力.md` |
| 官方 npm 扩展 | `08-懒猫生态/01-官方扩展.md` |
| 网络/VPN/白屏排障 | `09-常见问题/` |
| 入门阅读指引 | `01-欢迎/06-入门必看文集.md` |
| 移植合成速查 | `lazycat-porting/porting-handbook.md`（index id: `porting-handbook`） |

## 写作与维护约定

- 每篇文章有 frontmatter：`id`, `title`, `category`, `summary`, `tags`
- 新增/修改文章后应更新 `meta/index.json`
- 尽量保留原文，只做排版；内部链接用相对路径

## Codex / LPK 分发

本 Skill 可打包进 LPK `resources/skills/`（见 `06-专题/03-Skill MCP 规范.md`）：

- `lazycat-library/SKILL.md` + 手册正文 + `meta/index.json`
- `lazycat-porting/SKILL.md` + `porting-handbook.md`

Agent 同样遵循「先 index / handbook、后单篇」。

## 安装

本 skill 随 `lazycat-skills` 一起安装；手册正文需额外克隆 [lazycat-developer-manual](https://github.com/lazycat-cloud/lazycat-developer-manual) 或设置 `LAZYCAT_MANUAL_ROOT`。

```bash
npx skills add zhistor26/lazycat-skills
```

## 已知缺口（查询时提示用户）

- 图片资源 `public/`、`images/` 尚未放入仓库（文内 CDN 图片不受影响）
- 合成速查 `porting-handbook` 中 deploy-params 为截断摘要，以专栏原文为准
