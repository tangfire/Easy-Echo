# Auto TLS

这个例子演示如何自动从 Let’s Encrypt 获得 TLS 证书。 `Echo#StartAutoTLS` 接受一个接听 443 端口的网络地址。类似 `<DOMAIN>:443` 这样。

如果没有错误，访问 `https://<DOMAIN>` ，可以看到一个 TLS 加密的欢迎界面。

# 服务器

`server.go`

```go
package main

import (
    "net/http"

    "github.com/labstack/echo"
    "github.com/labstack/echo/middleware"
)

func main() {
    e := echo.New()
    // e.AutoTLSManager.HostPolicy = autocert.HostWhitelist("<your_domain>")
    // Store the certificate to avoid issues with rate limits (https://letsencrypt.org/docs/rate-limits/)
    // e.AutoTLSManager.Cache = autocert.DirCache("<path to store key and certificate>")
    e.Use(middleware.Recover())
    e.Use(middleware.Logger())
    e.GET("/", func(c echo.Context) error {
        return c.HTML(http.StatusOK, `
            <h1>Welcome to Echo!</h1>
            <h3>TLS certificates automatically installed from Let's Encrypt :)</h3>
        `)
    })
    e.Logger.Fatal(e.StartAutoTLS(":443"))
}

```

这段代码使用 **Go 的 Echo 框架**实现了一个自动启用 HTTPS 的 Web 服务，通过 **Let's Encrypt** 自动申请和管理 TLS 证书。以下是分步解析：

---

### **1. 初始化 Echo 实例**
```go
e := echo.New()
```
- **作用**：创建 Echo 框架的实例，用于管理路由、中间件和服务器配置。

---

### **2. 配置中间件**
```go
e.Use(middleware.Recover())
e.Use(middleware.Logger())
```
- **`Recover`**：捕获处理请求时的 `panic`，防止服务崩溃，返回 HTTP 500 错误。
- **`Logger`**：记录请求的详细信息（如请求方法、路径、响应状态码和耗时）。

---

### **3. 启用自动 TLS（关键部分）**
```go
// 注释部分为可选配置
// e.AutoTLSManager.HostPolicy = autocert.HostWhitelist("<your_domain>")
// e.AutoTLSManager.Cache = autocert.DirCache("<path to store key and certificate>")
e.Logger.Fatal(e.StartAutoTLS(":443"))
```
- **`StartAutoTLS(":443")`**：
    - **作用**：在 443 端口启动 HTTPS 服务器，并自动通过 Let's Encrypt 申请 TLS 证书。
    - **原理**：基于 ACME 协议（如 Let's Encrypt）自动完成证书颁发和续期，无需手动管理证书文件。
- **注释配置项**：
    - **`HostWhitelist`**：限制证书仅对特定域名生效（例如 `example.com`），避免滥用 Let's Encrypt 的速率限制。
    - **`DirCache`**：指定证书存储路径（如 `~/.cache/autocert`），避免重复申请证书。

---

### **4. 定义路由及响应**
```go
e.GET("/", func(c echo.Context) error {
    return c.HTML(http.StatusOK, `
        <h1>Welcome to Echo!</h1>
        <h3>TLS certificates automatically installed from Let's Encrypt :)</h3>
    `)
})
```
- **路由处理**：当访问根路径 `/` 时，返回一个 HTML 页面，提示 TLS 证书已自动配置。
- **`c.HTML()`**：发送 HTML 格式的响应，状态码为 200（OK）。

---

### **代码整体作用**
1. **自动 HTTPS**：服务启动后，首次访问时会自动向 Let's Encrypt 申请证书，后续请求均通过 HTTPS 加密。
2. **证书管理**：证书自动续期，无需人工干预（需确保服务域名可被公网访问且 443 端口开放）。
3. **安全性**：
    - 默认使用 TLS 1.3（若客户端支持），提升加密性能和安全性。
    - 通过 `HttpOnly` 和自动证书加密保障通信安全。

---

### **关键细节**
- **Let's Encrypt 限制**：免费证书需域名有效且服务器可被公网访问，同一域名每周最多申请 50 次。
- **适用场景**：适合需要快速部署 HTTPS 的公开服务（如个人博客、API 服务）。
- **本地测试限制**：若在本地运行，需配置域名解析（如修改 hosts 文件）并开放 443 端口，否则 Let's Encrypt 无法验证域名所有权。

---

### **扩展说明**
- **TLS 1.3 优势**：相比 TLS 1.2，握手时间从 2-RTT 缩短到 1-RTT，提升连接速度（Go 1.13+ 默认支持）。
- **证书存储**：建议通过 `DirCache` 持久化存储证书，避免服务重启后重复申请。
- **替代方案**：若需自签名证书（如内网环境），可参考 `crypto/tls` 生成证书并手动加载。

此代码是 Echo 框架实现 HTTPS 服务的简化方案，适合快速原型开发。生产环境中需补充域名白名单、错误监控和性能优化（如 HTTP/2 支持）。