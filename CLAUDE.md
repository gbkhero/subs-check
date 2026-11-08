# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概览

`subs-check` 是一个 Go 编写的订阅检测转换工具：拉取订阅链接 → 去重 → 测活 → 流媒体检测/重命名/过滤 → 测速 → 输出 `all.yaml`，并通过内置的 Sub-Store（内嵌 Node 二进制 + bundle）转换为多种客户端订阅格式。程序同时提供：

- `listen-port`（默认 `:8199`）：`/sub/*` 静态订阅、根路径兼容文件（`/all.yaml` 等）、`/export/:target`、Web 管理面板 `/admin`、`/admin/results` 与 `/api/*`。
- `sub-store-port`（默认 `:8299`）：仅绑定回环地址的 Sub-Store 子进程，用于订阅转换。

## 常用命令

```bash
# 构建 / 交叉编译（Makefile 变量会注入 Version，需在 git 仓库内执行）
make build
make build-all

# 运行（首次启动若配置文件不存在，会生成默认配置并直接退出）
go run . -f ./config/config.yaml

# 全部测试（CI 就是这一条）
go test ./...
# 单个测试
go test ./check -run TestPipeline_PreservesOrder -v
go test ./app -run TestRouter_ExportRoutes -v

# 格式化 / 静态检查（无 golangci-lint 配置）
gofmt -w .
go vet ./...
```

- 发布由 `.goreleaser.yaml` 在 `v*.*.*` tag 上完成；Docker 镜像由 `.github/workflows/release.yml` 构建。
- `Dockerfile` 在构建阶段从系统 node 生成 `assets/node_<os>_<arch>.zst`；本地构建依赖仓库中已提交的这些 zst 与 `sub-store.bundle.js.zst`。`assets/assets_<os>_<arch>.go` 用 build tags 选择对应二进制。
- 测试禁止真实网络访问；探针阶段可用 `SUB_CHECK_SKIP=1` 短路（见下）。

## 启动与配置生命周期

`main.go` → `app.New` → `app.Initialize` → `app.Run`：

1. `initConfigPath`：未传 `-f` 时配置位于**可执行文件目录**的 `config/config.yaml`。
2. `utils.SetCacheDir`：私有缓存目录为 `<配置文件目录>/cache/<配置文件名>`，`results.json` 与 `export/` 都在这里（不要放进公开的 output 目录）。
3. `app.loadConfig`：配置不存在时写入内嵌的 `config/config.example.yaml` 并 `os.Exit(0)`；新增配置字段必须同时加 `Config` 的 yaml tag 和示例配置。
4. `initResolver`：必须在任何代理拨号之前调用；`dns.enable=true` 时接管 mihomo 全局 resolver，`default-nameserver` 必须是纯 IP。
5. `initConfigWatcher`：fsnotify 监听配置目录并热重载。只有 `cron-expression` / `check-interval` 变化会重建定时器；端口、Web UI、sub-store 等启动期配置改动需重启。
6. `sub-store-port` 非空时后台启动内嵌 Sub-Store 子进程，崩溃后每 30 秒重启。

`config.GlobalConfig` 是全局可变单例，被多个 goroutine 直接读取。正因如此，流水线启动时会快照 `SpeedTestUrl`，避免运行中改配置把本轮测速打穿。

## 检测流水线（核心）

`App.checkProxies`：

1. `keep-days > 0` 时把 `save.LoadHistoryProxies()` 的历史节点前置到 `config.GlobalProxies`。
2. `proxy.GetProxies()` 并发拉取本地/远程订阅清单，解析 Clash YAML 或经 mihomo `convert.ConvertsV2Ray` 解析，按 `node-type` 过滤，并给每个节点写入 `sub_url` / `sub_tag`。
3. `proxy.DeduplicateProxies()` 按 `server:port:servername:password(uuid):sni:network` 去重。
4. `ProxyChecker.run` 驱动四阶段并发流水线，阶段间用 channel 连接：
   `dispatch → alive → media + filter → speed（可选）→ collector`。
   - 每个条目携带原始下标，collector 按订阅原序回填结果。
   - `success-limit` 命中或外部取消（SIGHUP / HTTP `/api/force-close`）会取消同一个 `context`；中间阶段 ctx-aware 丢弃，最后一段到 collector 的发送是无条件的，保证已判定通过的节点不丢。
   - 进度与各阶段计数是包级 atomic（`Progress`、`Available`、`MediaDone`、`FilterPassed`、`SpeedDone`、`SpeedOk`、`PhaseResults`），CLI 进度条和 `/api/status` 共用。
5. 每个探针通过 `check.CreateClient` 用 mihomo `adapter.ParseProxy` 构造独立 `http.Client`：自定义 `DialContext`、按节点统计字节数、全局限速 bucket；`watchdog` 会在远超所有超时后强制关闭卡死的底层 transport。
6. `checkMedia` 按 `platforms` 顺序并行检测，永不丢弃节点；`checkSpeed` 才按 `min-speed` 决定去留。

**测试模式**：`SUB_CHECK_SKIP=1` 让 alive/media/speed 全部直接通过，流水线单测依赖它；测试里用 `t.Setenv` + `check/render_test.go` 的 `withConfig` 还原全局配置，`app/router_test.go` 的 `newTestRouter` 用 `t.Cleanup` 还原配置与 `save.ResultsPath`。

## 命名、过滤与结果契约

- `check.RenderNameParts` 是唯一的节点展示名生成出口，格式为 `base|speed|media...|sub_tag`，纯函数。
- `rename-node=true` 时 base 被国家重命名**强覆盖**（`NodePrefix + flag + 国家_N`），避免上游已有的尾缀叠加。
- filter 在流媒体检测之后、测速之前执行，匹配的是**不含速度标签**的渲染名；`MatchesFilter`/`FilterResults` 不要改成直接读 `proxy["name"]`。
- `save.SaveConfig` 的顺序是隐式契约：先用原始 name 写 history → 再原地把 `proxy["name"]` 改成展示名并写 `results.json` → 最后序列化 `all.yaml`、推送 Sub-Store、拉取 `mihomo.yaml`/`base64.txt`、上传远程存储。调用后 `results` 视为已消费。

## 保存与导出

- `save/method` 提供 `SaveFunc` 实现：local / r2 / gist / webdav / s3；`save.SaveConfig` 通过 `save-method` 选择，并在非 local 时复用同一份字节。
- `save/results.go` 生成 `/admin/results` 的 `results.json`。`NodeRecord` 字段是白名单，凭据字段（password/uuid/private-key 等）绝不能加入。
- `export` 包管理按格式生成的订阅缓存：
  - 只有带 `X-API-Key` 的 `POST /api/export/:target` 才会触发 Sub-Store 转换，且是后台执行；公开的 `GET /export/:target` 只读已有文件。
  - 生成过的目标跨重启保留，每轮检测后由 `export.RoundComplete()` 自动重建，直到 `DELETE /api/export/:target`。
  - 文件名 `<id>.g<round>.<nano>`，新一轮覆盖旧的；preset 格式（mihomo/v2ray）直接从 output 目录由 `/sub/` 提供。
- `utils/substore.go` 负责与 Sub-Store 交互：写入名为 `sub` 的本地订阅，以及名为 `mihomo` 的 profile 文件（用 `customName: "subs-check专用,勿动"` 标记自己的 Script Operator）。更新使用 PATCH 浅合并，只发自己负责的字段，保留用户改动；接口对不存在的资源常返回 500，因此先看状态码再解析 JSON。

## Web 与运行环境

- `app/server.go` 注册所有路由；`newRouter` 与监听分离，因为 gin 路由冲突在注册时 panic，测试会捕获这一点。
- 管理 API 统一走 `authMiddleware`，校验 `X-API-Key`；`enable-web-ui=false` 时仍保留订阅静态路由与 `/export`。
- 模板与静态资源通过 `app/embed.go` 内嵌；访问日志写入 `$TMP/subs-check-gin.log`，应用日志写入 `$TMP/subs-check.log`，两者分离以免 HTTP 噪音污染 `/api/logs`。
- 常用环境变量：`API_KEY`（未配置 api-key 时的兜底）、`LOG_LEVEL`、`SUB_CHECK_SKIP`、`SUB_CHECK_MEM_LIMIT`、`SUB_CHECK_MEM_MONITOR`、`NODEBIN_PATH`、`SUB_STORE_PATH`、`MIHOMO_DEBUG`。
- `SUB_CHECK_MEM_LIMIT` 超限会启动新进程并退出当前进程，注意不要在校验/测试时设置。

## 代码风格与注意点

- 注释和用户可见日志以中文为主，保留现有风格；错误信息同样是中文。
- 全局状态较多（`config.GlobalConfig`、check 包的 atomic、`save.ResultsPath`、rename 计数器）。修改时优先沿用既有的包级变量与重置函数（如 `proxy.ResetRenameCounter`、`check.ResetPhaseResults`），不要在测试中泄漏状态。
- 取消语义：`check.RequestCancel` 只取消当前正在运行的流水线阶段，程序继续运行；`utils.SetupSignalHandler` 只监听 SIGHUP，不接管 SIGINT/SIGTERM。
- 不要提交 `config.yaml`、`config/cache/`、`output/` 或测试产物（`.gitignore` 已覆盖）；用 `-f` 指向其他路径时，其旁边的 `cache/` 同样是含节点凭据的运行数据。
- `config/config.example.yaml` 是用户文档的一部分，新增/修改配置项时同步更新其中的注释。
