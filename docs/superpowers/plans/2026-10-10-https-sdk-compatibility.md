# HTTPS 支持 — Implementation Plan

**Spec**: [2026-10-10-https-sdk-compatibility-design.md](../specs/2026-10-10-https-sdk-compatibility-design.md)
**Issue**: #54

---

## Overview

HTTPS listener 已在 main。本计划补齐两块：① SDK-facing 基础（`make dev-certs`、README、TLS 握手测试，已完成并在最新 main 上 rebase）；② reviewer 要求的**证书加载/更新/轮换**（热轮换组件 + 配置 + 测试，本次实现）。纯增量，向后兼容。

## Phase 1: 基础 — 已完成（`72fbc35`，rebase 自 `fcd27a3`）

- `internal/server/https_test.go`（新建，196 行）：
  - `generateSelfSignedCert(t, dir)` — crypto/x509 + ECDSA P-256 测试内生成，SAN `DNS:localhost, IP:127.0.0.1`，0600 权限写 `t.TempDir()`
  - `freePort(t)` — 预留临时端口
  - `TestServerHTTPSListener` — 起真实 `Server`（mock backend + `server.https` 配置）→ 仅信任该证书的 client 轮询 `GET /healthz` → 断言 TLS 握手完成 + 200 → `cancel()` + 优雅 `Stop()`
- `Makefile`：`dev-certs` target（openssl RSA 2048 / SHA256 / 365 天，`/CN=localhost`，SAN `DNS:localhost,IP:127.0.0.1` → `bin/certs/`，已被 `bin/` gitignore 规则覆盖）

## Phase 2: 文档 — 已完成（`72fbc35`）

- `README.md` HTTPS 段落扩展：`make dev-certs` 工作流、SDK 默认行为开箱即用（无需 `debug=True`）、自签证书的信任方式 / `E2B_SKIP_SSL_VERIFY=1`

## Phase 4: 证书热轮换 — 本次实现

### 4.1 配置

- `internal/config/config.go`：`HTTPSConfig` 加 `ReloadInterval time.Duration`（`mapstructure:"reloadInterval"`）
- `internal/config/config.go` defaults：`v.SetDefault("server.https.reloadInterval", "30s")`
- `configs/e2bgateway-example.yaml`：`server.https.reloadInterval` 注释示例

### 4.2 组件

- `internal/server/certreload.go`：
  - `certReloader{certFile, keyFile string; interval time.Duration; current atomic.Pointer[tls.Certificate]; modTime/size snapshot}`
  - `newCertReloader(certFile, keyFile string, interval time.Duration) (*certReloader, error)` — 构造即加载校验（fail-fast）
  - `GetCertificate(*tls.ClientHelloInfo) (*tls.Certificate, error)` — 返回当前证书
  - `watch(ctx context.Context)` — ticker 轮询；mtime+size 变化才 reload；失败保留旧证书 + warn；ctx 取消退出
  - `reload() error` — `tls.LoadX509KeyPair` → 原子替换 → info 日志（notAfter）
- `internal/server/http.go`：
  - 创建 httpsServer 时构建 reloader（错误向上返回，fail-fast）
  - `s.httpsServer.TLSConfig = &tls.Config{GetCertificate: reloader.GetCertificate, MinVersion: tls.VersionTLS12}`
  - `Start()`：ctx 传入 `go reloader.watch(ctx)`；`ListenAndServeTLS("", "")`（证书由 GetCertificate 提供）
  - `Server` 结构体保存 reloader 引用（watch 生命周期）

### 4.3 测试（TDD：先写测试）

- `internal/server/certreload_test.go`：
  - 初次加载成功 / 文件不存在 fail-fast
  - 替换证书文件后（interval=20ms，轮询等待）`GetCertificate` 返回新 serial
  - 主证书文件替换为非法内容 → 保留旧证书，`GetCertificate` 正常
  - `interval=0` → 不启动轮询（替换文件后 serial 不变）
  - 并发 `GetCertificate` + reload（`go test -race`）
- `internal/server/https_test.go` 增加：
  - `TestServerHTTPSListenerCertRotation` — 起真实 Server（短 reloadInterval）→ 握手拿 serial A → 原子替换 cert/key 文件 → 等待轮换 → 新握手拿 serial B
- 注意：证书文件名/路径不变（模拟 K8s Secret 更新），测试内用 `os.Rename` 或直接覆盖写

### 4.4 验证

- `go build ./...` / `go build -tags=e2e ./...`
- `go vet ./...`
- `go test -race ./internal/server/ ./internal/config/`
- 全量 `go test -race ./internal/... ./test/...`
- golangci-lint：新文件 0 findings（已知 Windows gci CRLF 误报除外）
- `go mod tidy` 零 diff

## Phase 5: Follow-ups

- Q1 Helm TLS secret volume + `server.https` values 装配
- Q2 http→https 重定向中间件（可选）
- Q3 https E2E（Kind 内 cert-manager 签发）

## Checklist

- [x] Spec written（2026-11-11 更新：证书热轮换设计）
- [x] Phase 1/2 基础实现（`72fbc35`）
- [ ] Phase 4 配置 + 组件 + 接线
- [ ] Phase 4 测试（单测 + 轮换 e2e + `-race`）
- [ ] 全量验证通过
- [ ] 提交（DCO `Signed-off-by: Windswithyou`）
- [ ] 推送 fork，开 PR（`Closes #54`）
