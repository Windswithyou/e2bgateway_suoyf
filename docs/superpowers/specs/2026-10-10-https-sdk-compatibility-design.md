# HTTPS 支持：TLS 终结与证书热轮换

**Issue**: #54 — [M1-R1.1][代码工程化] e2bgateway 支持 https
**Date**: 2026-10-10（2026-11-11 按 reviewer 意见扩展证书管理）
**Priority**: P1（SDK 协议兼容性关键路径）
**Approach**: B — HTTPS listener + **证书热轮换**（reviewer 2026-11-09 要求：Router 终结 HTTPS，配套证书加载/更新/轮换，后端明文 HTTP）
**实现状态**: 基础部分已实现（`feat/https-support` @ 重新基于最新 main 的 `72fbc35`）；证书热轮换实现中

---

## 1. Problem Statement

E2B Python/JS SDK 在非 debug 模式下**默认用 https** 连接 envd（URL 形如 `https://{port}-{sandboxID}.{envdDomain}`）。网关若只监听 http，用户必须显式传 `debug=True` 才能工作——违背"标准 E2B SDK 开箱即用"的项目定位。

上游 main 已具备基础 HTTPS listener，但对照 reviewer 要求（TLS 卸载 + 证书加载/更新/轮换）仍有缺口：

| 项 | 状态 |
|---|---|
| HTTPS 监听器（`server.https` 三字段齐备才启用） | ✅ 已在 main（`internal/server/http.go`） |
| `HTTPSConfig{Address, CertFile, KeyFile}` 配置结构 | ✅ 已在 main（`internal/config/config.go`） |
| `configs/e2bgateway-example.yaml` 的 `server.https` 示例 | ✅ 已有 |
| 证书加载 | ✅ 启动时 `ListenAndServeTLS(certFile, keyFile)` 一次性加载 |
| **证书更新/轮换（不重启）** | ❌ 缺口——换证书必须滚动重启 |
| reload 可观测性 | ❌ 缺口——轮换无日志 |
| 本地开发证书生成 | ✅ `make dev-certs`（分支已实现） |
| README HTTPS 部署指南 | ✅ 分支已补 |
| HTTPS listener 测试覆盖 | ✅ 基础握手测试（分支已实现）；轮换测试本次补 |

## 2. Design Goals

- SDK 默认 https **开箱即用**：配置 `server.https` 后无需 `debug=True`
- **证书热轮换**：cert-manager/Secret 挂载文件更新后，gateway 在 `reloadInterval` 内自动热加载，**不重启、不断连**
- 失败安全：坏证书/半写入状态**保留旧证书继续服务**，仅告警，下轮重试
- 本地开发**一条命令**出证书：`make dev-certs`
- HTTPS listener 有**真实 TLS 握手**与**证书轮换**的单测覆盖
- 纯增量改动，向后兼容：不配置 `server.https` 行为完全不变

**非目标（本迭代明确不做，均为 follow-up）**：

- ❌ Helm chart TLS secret volume + `server.https` 装配（issue/reviewer 未要求）
- ❌ ACME/autocert 自动证书签发
- ❌ 强制 http→https 重定向中间件
- ❌ SIGHUP 手动触发轮换（轮询已覆盖）
- ❌ https E2E（Kind 内 cert-manager 签发）
- ❌ gateway→envd 内部连接 TLS（pod 内网，非 SDK-facing 路径）

## 3. Architecture

### 3.1 接口变化

无。`SandboxAdapter` 接口不变，所有 adapter 不受影响。

### 3.2 服务行为

`server.Start()` 双监听：http 与 https 共享**同一个 chi router**，中间件链、路由、envd proxy 完全一致。TLS 只在传输层终结，后端 envd 走明文 HTTP。

WebSocket：chi 的 upgrade 处理与传输层无关，已验证 `internal/server`、`internal/streaming`、`internal/api` 无硬编码 `http://` 破坏 wss 的构造——**wss 经 https listener 自动可用，无需改动**。

### 3.3 证书热轮换组件

新增 `internal/server/certreload.go`：

```
certReloader
├── certFile, keyFile string          # 证书文件路径
├── interval time.Duration            # 轮询间隔（0 = 禁用，仅启动加载）
├── current atomic.Pointer[tls.Certificate]
├── load() error                      # tls.LoadX509KeyPair + 原子替换
├── watch(ctx)                        # 后台轮询 mtime+size，变化即 load()
└── GetCertificate(*tls.ClientHelloInfo) (*tls.Certificate, error)
```

- **启动时**：构造即加载并校验；失败返回错误（fail-fast，避免带病启动）
- **运行中**：goroutine 每 `interval` 比较 cert/key 的 mtime+size；发现变化才重新加载
- **原子替换**：`atomic.Pointer[tls.Certificate]`，握手并发安全；已建立连接不受影响
- **失败安全**：加载失败（文件半写入/内容非法）保留旧证书，`slog.Warn` 记录，下轮重试
- **生命周期**：watch goroutine 随 `Server.Start` 的 ctx 取消退出

接线（`internal/server/http.go`）：

```go
reloader, err := newCertReloader(certFile, keyFile, cfg.Server.HTTPS.ReloadInterval)
// ...
s.httpsServer.TLSConfig = &tls.Config{
    GetCertificate: reloader.GetCertificate,
    MinVersion:     tls.VersionTLS12,
}
go reloader.watch(ctx)
// 启动：ListenAndServeTLS("", "") —— 证书由 GetCertificate 提供
```

### 3.4 数据流

```
E2B SDK (默认 https, 无需 debug=True)
       │
       ▼
https://{port}-{sandboxID}.{envdDomain}   ← TLS 由 gateway https listener 终结
       │                                     （证书可热轮换）
       ▼  同一 chi router / envd proxy（Host 头路由）
envd (http, pod 内网 49983)
```

## 4. API Surface

无新端点。`/healthz`、`/readyz`、`/sandboxes/*` 等在两个 listener 上等价可用。

## 5. Configuration

```yaml
server:
  https:                          # address/certFile/keyFile 齐备才启用 HTTPS listener
    address: "0.0.0.0:8443"
    certFile: "/etc/e2bgateway-tls/tls.crt"
    keyFile: "/etc/e2bgateway-tls/tls.key"
    reloadInterval: "30s"         # 证书文件轮询间隔，默认 30s；0 = 禁用热轮换
```

- 任一 address/certFile/keyFile 缺失 → HTTPS listener 不创建，仅 http（向后兼容的关键设计）
- `reloadInterval` 默认 `30s`（`v.SetDefault("server.https.reloadInterval", "30s")`）；`0` 时仅启动加载
- Viper/mapstructure 对 mapstructure tag 大小写不敏感，`certfile`/`certFile` 均可解析

**Makefile target（分支已实现）**：

```makefile
dev-certs: ## Generate a self-signed TLS certificate for local HTTPS development.
	@mkdir -p bin/certs
	openssl req -x509 -newkey rsa:2048 -sha256 -days 365 -nodes \
		-keyout bin/certs/tls.key -out bin/certs/tls.crt \
		-subj "/CN=localhost" \
		-addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

## 6. Security Considerations

- **密钥不入库**：`bin/certs/` 落在已有 `bin/` gitignore 规则内，不会误提交
- **测试证书自包含**：单测内 `crypto/x509` + ECDSA P-256 现场生成，写 `t.TempDir()`（0600 权限），24h 有效期，不提交任何证书文件
- **dev-certs 定位**：RSA 2048 / 365 天自签，**仅限本地开发**；生产必须用 cert-manager / 正规 CA（README 明确）
- **不记录密钥**：无任何密钥内容进日志；reload 日志只含路径与 notAfter
- **失败安全**：轮换失败保留旧证书，不中断服务、不降低 TLS 版本
- **最小 TLS 版本**：显式 `MinVersion: tls.VersionTLS12`
- **威胁模型**：TLS 终结在 gateway，与 K8s ingress TLS 终结是二选一关系；内网 envd 路径信任边界不变

## 7. Observability

- HTTPS 启动打印与 HTTP 对齐（`E2BGateway HTTPS starting on %s`）
- 轮换成功：`slog.Info("TLS certificate reloaded", certFile, notAfter)`
- 轮换失败：`slog.Warn("TLS certificate reload failed; keeping previous certificate", err)`
- 无新指标（日志足够，YAGNI）

## 8. Testing Strategy

| 层 | 覆盖 | 状态 |
|---|---|---|
| 单测 `internal/server/certreload_test.go` | 初次加载；文件替换后 serial 变化；坏证书保留旧证书；`interval=0` 不重载；并发 `GetCertificate`（`-race`） | 待实现 |
| 单测 `internal/server/https_test.go` | `TestServerHTTPSListener` 基础握手（已有）；`TestServerHTTPSListenerCertRotation`：起真实 Server → 握手取 serial A → 替换证书文件（短 interval 50ms）→ 新握手取 serial B | 待实现（基础已 PASS） |
| 单测回归 | 全量 `go test ./internal/... ./test/...` + `-race` | 待跑 |
| lint | 新文件 0 findings（goheader/godot/revive/modernize 干净） | 待跑 |
| E2E（https/wss over Kind） | 需要 Kind 内证书签发，复杂度不成比例 | ⏸ follow-up |

## 9. Migration / Rollout

- **纯增量**：不配置 `server.https` → 行为与现状完全一致（HTTPS listener 不创建）
- 已配置 `server.https` 的部署：`reloadInterval` 默认开启（30s），行为增强但兼容；`0` 可回到旧行为
- 无 feature flag；配置即启用
- 生产部署路径（Helm/ingress TLS）不受影响

## 10. Open Questions

| # | 问题 | Owner | 状态 |
|---|---|---|---|
| Q1 | Helm TLS secret volume + `server.https` 装配 | 待定 | follow-up（issue/reviewer 未要求） |
| Q2 | 生产是否需要 http→https 重定向中间件？ | 待定 | follow-up（SDK 默认 https，非必需） |
| Q3 | https E2E（Kind 内 cert-manager 签发）是否纳入 M1？ | 待定 | follow-up |
