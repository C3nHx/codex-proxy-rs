# 运行时子路径与在线更新验收

目标是在 `/cprs/` 下复用官方发行物，并在整体更新和回退后保留部署前缀。

实现基于上游 `9d2aad95ece02c34634cf19e84cec1244d2bb21c`。前缀由 API 配置中的 `WebBasePath` 校验和持有。静态 HTML 响应在内存中应用 `<base>`，前端统一从 `document.baseURI` 取得 Router 和 API 前缀。更新器、发行校验、文件指纹和整组回退逻辑没有放宽。

最终代码提交：[cf829f645eab540ece01e298784e658f8d9cb563](https://github.com/C3nHx/codex-proxy-rs/commit/cf829f645eab540ece01e298784e658f8d9cb563)。最终目录布局修正仅将生产和测试叶子模块改为 `web.rs`，没有改变演练已验证的逻辑；Linux 格式、Clippy、670 项 API 测试和 16 项架构检查均在最终树上重新通过。

## 真实隔离部署

环境使用本机 Windows 网关、独立 PostgreSQL 18 和 Redis 8 容器、Caddy 2.11.9，以及 Edge 浏览器。应用、数据、端口和管理员均为测试专用，没有导入生产账号或凭据。

两份发行包由同一工作区源码构建，版本元数据分别为 `3.50.0` 和 `3.50.1`。这些是本地模拟发行物，不是已发布的官方版本。两份包复用同一次前端构建，仅增加不同的测试版本标记文件。发布索引和下载地址由回环 HTTP 服务提供；网关、认证、下载校验、更新、重启与回退均执行实际产品代码。

演练使用实际接口完成以下步骤：

1. 经 Caddy 的 `/cprs/` 入口登录，核对 session Cookie 的 `/cprs/` 和 refresh Cookie 的 `/cprs/api/auth` 路径
2. 通过系统检查接口发现模拟下一版，下载平台归档与 `checksums.txt`，校验 SHA-256 和发行清单，完成整组安装
3. 通过管理接口自重启，实际运行版本变为 `3.50.1`
4. 通过管理接口回退并自重启，实际运行版本恢复 `3.50.0`

每个阶段均读取运行版本和发行标记，验证原登录会话仍有效。浏览器实际登录并直接打开、刷新 `/cprs/accounts`，检查 PWA 地址和前缀。三个阶段的磁盘 `index.html` SHA-256 均为 `b5763f8e6f359fdfe41a1b9645971e318306b89ec9a29388d06cd88f8a0e3d7d`，仍保留根路径 `<base>`；实际 HTML 响应使用 `/cprs/`。结果说明部署配置没有写入或改动发行文件。

真实演练通过，测试网关、Caddy 和数据库服务已停止。没有修改生产部署，也没有调用真实模型。

## 根路径与子路径对照

未修改上游的默认前端在根路径可以加载，在仅提供 `/cprs/` 的入口下无法加载。修改后的同一前端目录分别在 `/`、`/cprs/` 和 `/nested/cprs/` 下加载并刷新成功，静态资源和 manifest 均保留前缀。该对照使用真实浏览器，完整升级回退则使用上节的真实网关部署。

## 质量检查

- Linux Node 26.13.0、pnpm 12.6.0 下，未修改基线与修改后前端均通过锁文件安装、全量 `format:check` 和生产构建
- Rust 1.99.1 的 `cargo fmt --all --check` 通过
- `gateway-api` 的严格 Clippy、全部 target 与 feature 通过
- 三项新增回归测试通过，覆盖配置兼容、前缀输入、HEAD、静态资源 Range、发行目录替换及回退
- Windows 全量 API 测试为 669 通过、1 失败。同一实体 socket 背压测试在未修改上游上也失败，不能计为通过
- Windows 全 workspace Clippy 的六项 `result_large_err` 位于未修改 OpenAI 模块，在未修改上游上也复现
- Windows 架构检查为 8 通过、8 失败，错误涉及 Cargo 路径解析；这些检查没有修改，另在 Linux 环境运行同项检查
- 隔离、断网 Linux 容器中的 Rust 1.99.1 全 workspace 严格 Clippy 通过，覆盖全部 target 和 feature；完整 `gateway-api` 套件 670 项全部通过
- Linux workspace 架构检查 16 项全部通过，生产与测试叶子模块均使用 `web.rs`

没有改写协作规则、工作流或迁移文件。保留了前项维护者提交中的登录恢复修正，使用 `router.options.history.location` 避免重复前缀。第一项 PR 的原 CI 被后续提交取消，最终 CI 和 Security 均成功，不能表述为原 PR 留有失败 CI。

## 验证边界

本次完整演练使用 HTTP 回环入口和模拟发布服务，没有访问真实模型服务。反向代理继续剥除请求前缀并重写 Cookie 和站内 Location。已有构建时前缀部署需要先切换到包含本实现的完整发行物，再沿用后续在线更新。
