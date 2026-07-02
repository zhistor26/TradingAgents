# 提审前检查清单

移植完成后、执行 `appstore publish` 前逐项确认。

## A. 配置文件

- [ ] `package.yml`：`package`、`version`、`name`、`description` 已填写
- [ ] `package.yml`：`locales` 含 `zh-CN` 与 `en-US`（或目标语言）
- [ ] `lzc-manifest.yml`：routes / services / binds 与 Docker 来源一致
- [ ] `lzc-build.yml`：`pkgout`、`icon: ./icon.png` 已配置
- [ ] `icon.png` 存在且尺寸合适（商店 logo）
- [ ] 持久化路径均在 `/lzcapp/var` 或 `/lzcapp/cache` 下

## B. 本地发布包验证（必做）

```bash
lzc-cli project build
lzc-cli lpk info ./<package>-<version>.lpk
lzc-cli lpk install ./<package>-<version>.lpk
lzc-cli project info
```

- [ ] `lpk install` 无报错
- [ ] 启动器可打开应用，页面非白屏
- [ ] 重启应用后数据仍在（`/lzcapp/var`）
- [ ] 启动与首屏加载 < 5 分钟

## C. 镜像

- [ ] LPK 内引用的镜像在审核环境可拉取
- [ ] 或已 `lzc-cli appstore copy-image` 并更新 manifest 中的 `image` 后重新 build
- [ ] 或已通过 `lzc-build.yml` 的 `images` 内嵌到 LPK

## D. 审核硬性要求

- [ ] **免密登录**已配置（inject 或 OIDC，见 `06-专题/01-免密登录.md`）
- [ ] 应用名称、描述、使用说明支持多语言（`locales`）
- [ ] 商店截图等资源在 publish / 开发者中心流程中准备就绪

## E. 账号与提交

- [ ] 已在 https://developer.lazycat.cloud/manage 完成开发者申请
- [ ] 已阅读 `01-欢迎/05-应用上架审核指南.md`
- [ ] `lzc-cli --version` ≥ 1.2.54

```bash
lzc-cli appstore publish ./<package>-<version>.lpk
```
