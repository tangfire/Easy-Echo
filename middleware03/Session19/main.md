# Session (会话) 中间件

Session 中间件促进了 gorilla/sessions 支持的 HTTP 会话管理。默认提供了基于 cookie 与文件系统的会话存储；然而，你也可以访问 community maintained implementation 来参考其各式各样的后端实现。

`Echo 社区贡献`


# 依赖

```go
import (
  "github.com/gorilla/sessions"
  "github.com/labstack/echo-contrib/session"
)
```

用法

```go
e := echo.New()
e.Use(session.Middleware(sessions.NewCookieStore([]byte("secret"))))

e.GET("/", func(c echo.Context) error {
  sess, _ := session.Get("session", c)
  sess.Options = &sessions.Options{
    Path:     "/",
    MaxAge:   86400 * 7,
    HttpOnly: true,
  }
  sess.Values["foo"] = "bar"
  sess.Save(c.Request(), c.Response())
  return c.NoContent(http.StatusOK)
})
```
这段代码使用 **Go 的 Echo 框架**实现了一个带有会话（Session）管理的简单 Web 服务。以下是分步解释：

---

### **1. 创建 Echo 实例**
```go
e := echo.New()
```
- **作用**：初始化一个 Echo 框架的 Web 服务实例，用于处理 HTTP 请求和路由。

---

### **2. 注册会话中间件**
```go
e.Use(session.Middleware(sessions.NewCookieStore([]byte("secret"))))
```
- **会话中间件**：使用 Cookie 存储会话数据，需配合中间件（例如 `github.com/ipfans/echo-session` 或类似库）。
- **`sessions.NewCookieStore`**：创建一个基于 Cookie 的会话存储，`secret` 是用于加密和签名 Cookie 的密钥（需保密）。
- **作用**：所有请求都会经过此中间件，自动解析或创建会话。

---

### **3. 定义根路径路由**
```go
e.GET("/", func(c echo.Context) error {
  // 会话操作逻辑
})
```
- **`e.GET("/", ...)`**：处理根路径的 GET 请求。
- **处理函数**：当访问 `GET /` 时，执行此函数。

---

### **4. 获取或创建会话**
```go
sess, _ := session.Get("session", c)
```
- **`session.Get("session", c)`**：获取名为 "session" 的会话对象。若不存在，则自动创建。
- **空白标识符 `_`**：此处忽略了错误（实际代码建议处理错误）。

---

### **5. 配置会话 Cookie 参数**
```go
sess.Options = &sessions.Options{
  Path:     "/",       // Cookie 生效路径（根路径）
  MaxAge:   86400 * 7, // 过期时间：7 天（单位：秒）
  HttpOnly: true,      // 仅允许 HTTP 访问，防止 JavaScript 操作
}
```
- **`Path`**：限制 Cookie 仅在指定路径下发送。
- **`MaxAge`**：会话有效期（7 天后自动过期）。
- **`HttpOnly`**：增强安全性，防止 XSS 攻击窃取 Cookie。

---

### **6. 存储数据到会话**
```go
sess.Values["foo"] = "bar"
```
- **`sess.Values`**：以键值对形式存储会话数据（此处 `foo` 的值为 `bar`）。
- **注意**：Cookie 存储有大小限制（通常 4KB），且敏感数据需加密。

---

### **7. 保存会话到响应**
```go
sess.Save(c.Request(), c.Response())
```
- **作用**：将当前会话数据保存到 HTTP 响应中，通过 Set-Cookie 头将加密后的会话 ID 或数据发送给客户端。

---

### **8. 返回响应**
```go
return c.NoContent(http.StatusOK)
```
- **`c.NoContent(http.StatusOK)`**：返回 HTTP 状态码 200，无响应内容。

---

### **代码整体作用**
1. **初始化服务**：创建一个 Echo Web 服务。
2. **会话管理**：通过 Cookie 存储会话数据，确保后续请求能识别同一用户。
3. **数据存储**：在会话中存储键值对 `foo: bar`，客户端再次访问时可通过会话读取该数据。
4. **安全性**：通过 `HttpOnly` 和 `secret` 密钥防止 Cookie 篡改或窃取。

---

### **关键细节**
- **会话存储方式**：使用 Cookie 存储，数据经过加密和签名，但**敏感数据仍需避免存储**（如密码）。
- **密钥管理**：`secret` 需替换为随机强密钥，且不应硬编码在代码中（可通过环境变量注入）。
- **错误处理**：代码中忽略了 `session.Get` 的错误，实际需处理（例如 `if err != nil { ... }`）。

---

### **示例流程**
1. 用户首次访问 `GET /`，服务创建一个新会话，存储 `foo: bar`，返回包含会话 Cookie 的响应。
2. 用户再次访问时，浏览器自动发送 Cookie，服务可读取会话中的 `foo` 值。

---

### **扩展场景**
- **读取会话**：在后续请求中，可通过 `session.Get` 获取 `sess.Values["foo"]` 的值。
- **其他存储**：若需存储大量数据，可改用服务端存储（如 Redis、数据库），仅通过 Cookie 保存会话 ID。

这段代码是会话管理的基础实现，适用于用户身份跟踪、临时数据存储等场景。


# Custom Configuration

用法


```go
e := echo.New()
e.Use(session.MiddlewareWithConfig(session.Config{}))
```

# 配置

```go
Config struct {
  // Skipper defines a function to skip middleware.
  Skipper middleware.Skipper

  // Session store.
  // Required.
  Store sessions.Store
}
```

默认配置

```go
DefaultConfig = Config{
  Skipper: DefaultSkipper,
}

```