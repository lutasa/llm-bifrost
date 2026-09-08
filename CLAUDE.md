# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目身份与工作区上下文

- 本仓库是 **`maximhq/bifrost` 的 fork**：remote `github.com/lutasa/llm-bifrost`，工作分支 `dev`。文档/注释中残留的 "bifrost" 字样均指本仓库
- **改任何代码前，先读完同目录 `AGENTS.md`**（53KB）：仓库布局、请求流、Provider/Plugin 模式、19 条 Gotchas、新增 Provider checklist、前端规范 —— 本文件不重复其内容，两者冲突时以 `AGENTS.md` + 实际代码为准
- 相邻目录 `../llm-gateway/` 是编译为 `.so` 动态加载的 **LLMPlugin**（安全插件），锁定本仓库的 `bifrost/core v1.5.15`，部署目标 Bifrost v1.5.7。**改 `core/schemas/` 导出类型或插件接口签名 → 必须验证 `../llm-gateway/` 仍能编译**（`cd ../llm-gateway && go build ./...`）

## 多模块工作区（新会话必读）

- 根目录**默认没有 go.work**，先 `make setup-workspace` 生成
- `go mod tidy` 必须在具体模块目录内执行（`core/`、`framework/`、`transports/`、`plugins/<name>/`、`cli/`），不要在根目录跑
- Go 要求 **1.26.1+**
- 热路径技术选型：**fasthttp**（非 net/http，唯一例外 Bedrock 走 net/http+SigV4+HTTP2）、**sonic**（非 encoding/json）、单字段 JSON 操作用 gjson/sjson（`providerUtils.GetJSONField` 等封装），回写必须 `providerUtils.MarshalSorted`

## 命令速查（在仓库根目录）

```bash
make dev                                 # UI + API 热重载（air）
make build                               # 构建 bifrost-http
make test-core PROVIDER=openai TESTCASE=TestSimpleChat   # provider 测试（live API，花钱）
make test-core PROVIDER=openai PATTERN=Stream            # 按子串匹配
make test-core DEBUG=1                   # Delve 调试 :2345
make test-mcp TESTCASE=TestAgentLoop     # MCP 测试（mock，不花钱）
make test-framework                      # 需先: docker compose -f tests/docker-compose.yml up -d
make run-e2e FLOW=providers              # Playwright E2E（需 dev server）
make lint && make fmt
```

⚠️ provider 测试**只用 `make test-core`**，裸 `go test ./core/providers/...` 会跳过 llmtests 场景（流式/工具调用回归抓不到）。

## 功能模块地图（全局定位用）

| 模块 | 位置 | 一句话职责 |
|------|------|-----------|
| M1 推理引擎 | `core/`（bifrost.go、inference.go、providers/、schemas/、pool/、keyselectors/） | 请求调度、Provider 隔离队列、fallback/retry、加权随机 key 选择、全部共享类型 |
| M2 MCP 网关 | `core/mcp/` + `transports/bifrost-http/handlers/mcp*.go` + `framework/mcpcatalog/` | Agent 多轮工具调用循环、MCP 客户端/服务器双角色、工具 4 层过滤、MCP OAuth2 |
| M3 HTTP 网关 | `transports/bifrost-http/`（handlers/lib/integrations）+ `config.schema.json` | 60+ 端点、中间件链、7 大 SDK drop-in 兼容、Realtime(WS/WebRTC)、Skills、Webhooks |
| M4 插件体系 | `core/schemas/plugin.go`（接口）+ `plugins/*`（12 个内置） | LLMPlugin/MCPPlugin/HTTPTransportPlugin/ObservabilityPlugin 四接口；`../llm-gateway` 是外挂 LLMPlugin（.so） |
| M5 框架服务 | `framework/*`（24 包） | configstore/logstore/vectorstore/objectstore、streaming 累积器、gencache/lrucache、batchaccounting、sidekiq 任务队列、oauth2/temptoken、migrator |
| M6 Web UI | `ui/`（React+Vite+TanStack Router+RTK Query） | 可视化配置/监控/治理；`data-testid` 是 E2E 承重属性 |
| M7 质量保障 | `core/internal/llmtests|mcptests/`、`tests/e2e/`、`tests/integrations/` | live 场景测试、mock 测试、Playwright、SDK 集成、provider-harness.json（wire 级 pin） |
| M8 分发运维 | `cli/`、`npx/`、`helm-charts/`、`terraform/`、`recipes/`、`docs/` | CLI、npx 启动、K8s/IaC、Mintlify 文档 |

## 排查高频入口（症状 → 位置）

| 症状 | 先看 |
|------|------|
| 请求排队/调度/超时异常 | `core/bifrost.go`（ProviderQueue、ChannelMessage）、`core/inference.go`（fallback 触发逻辑） |
| 某 provider 响应/报文异常 | `core/providers/<name>/`：`errors.go`（错误映射）、`chat.go`/`responses.go`（转换器） |
| 流式被 30s 掐断 / chunk 丢失 | provider 是否漏用 `streamingClient`；`framework/streaming/`（accumulator、gate） |
| 插件不生效 / 行为顺序怪 | 注册顺序语义（pre 正序、post 逆序）；动态 `.so` 加载在 `transports/bifrost-http/server/plugins.go` 的 `loadCustomPlugin` |
| fallback 后计数/计费翻倍 | **已知语义**：fallback 重放整个插件流水线（Gotcha #9），非 bug |
| MCP 工具不出现 / 被过滤 | 4 层过滤链（全局→client→tool→请求级 header），`core/mcp/toolmanager.go` |
| 治理/预算/限流误判 | `plugins/governance/`、`handlers/governance.go`、`framework/queryscope/` |
| 配置字段不识别 / 文档与实际不符 | `transports/config.schema.json` 是唯一真源（~2700 行） |
| 内存泄漏 / 对象池问题 | `core/pool/`（`-tags pooldebug` 构建）+ `handlers/devpprof.go` 诊断端点 |
| BifrostContext 值 mysteriously 丢失 | 保留键被 `BlockRestrictedWrites()` **静默丢弃**（Gotcha #11）；流级大数据禁止进 ctx |
| 改了 OpenAI provider，别家也坏了 | **预期级联**：9+ OpenAI-compatible provider 委托 openai 实现（Gotcha #6） |
| API 层任何问题（401/路由/中间件/端点清单） | 详文档：`../docs/llm/llm-bifrost-API层分析.md`（280 路由全景 + 中间件链 + 双轨认证 + 排查表） |

## 工作区级提醒

- 本文件（CLAUDE.md）在 fork 中为 untracked 文件，注意不要误提交到上游 fork 分支
- `../CLAUDE.md`（工作区根）记录两项目的耦合关系与部署链，涉及 llm-gateway 联动时先读它
