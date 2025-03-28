# 错误处理

## 错误处理程序

Echo 提倡通过中间件或处理程序 (handler) 返回 HTTP 错误集中处理。集中式错误处理程序允许我们从统一位置将错误记录到外部服务，并向客户端发送自定义 HTTP 响应。

你可以返回一个标准的 `error` 或者 `echo.*HTTPError`。

例如，当基本身份验证中间件找到无效凭据时，会返回 401未授权错误 (401-Unauthorized)，并终止当前的 HTTP 请求。

```go
e.Use(func(next echo.HandlerFunc) echo.HandlerFunc {
        return func(c echo.Context) error {
            // Extract the credentials from HTTP request header and perform a security
            // check

            // For invalid credentials
            return echo.NewHTTPError(http.StatusUnauthorized)

            // For valid credentials call next
            // return next(c)
        }
    })
}
```

你也可以不带消息内容调用 `echo.NewHTTPError()`，这种情况下状态文本会被用作错误信息，例如 `Unauthorized`。


---

这段代码是 **Eolang Echo框架中一个未完成的认证中间件实现**，其核心逻辑是拦截所有请求并进行身份验证，但当前代码处于“仅拒绝请求”的演示状态。以下是分层次解析：

---

### **1. 代码结构解析**
```go
e.Use(func(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        // 提取请求头中的凭证并进行安全检查（未实现具体逻辑）
        // 直接返回未授权错误，中断后续处理
        return echo.NewHTTPError(http.StatusUnauthorized)
        // 若凭证有效，应调用 next(c) 继续执行后续中间件或路由处理
    }
})
```
- **代码功能**：
    - 通过`e.Use()`注册一个全局中间件。
    - 中间件内部未实际实现凭证提取与验证逻辑，直接返回`HTTP 401 Unauthorized`错误，导致所有请求被拒绝。
    - 注释中提示了两种分支逻辑：验证失败返回错误，验证成功则调用`next(c)`传递请求。

---

### **2. 关键机制**
#### **（1）中间件的链式调用**
- **正常流程**：中间件通过`return next(c)`将控制权交给下一个中间件或路由处理器。
- **中断流程**：若中间件直接返回错误（如本例），Echo会跳过后续中间件和路由处理，直接向客户端返回错误响应。

#### **（2）HTTP错误处理**
- `echo.NewHTTPError()`用于创建标准HTTP错误响应，本例中固定返回401状态码，实际应用需根据验证结果动态决定是否返回此错误。

#### **（3）中间件作用域**
- 此中间件注册在根层级（`e.Use()`），因此对所有路由生效。若需限定作用范围，应使用组级（`e.Group()`）或路由级中间件。

---

### **3. 实际应用场景（需补充的逻辑）**
#### **（1）凭证提取与验证**
需在中间件中添加以下逻辑（示例为基本认证）：
```go
// 从请求头提取BasicAuth凭证
username, password, ok := c.Request().BasicAuth()
if !ok {
    return echo.NewHTTPError(http.StatusUnauthorized, "Missing credentials")
}
// 验证凭证（示例为硬编码，实际应查询数据库或缓存）
if username != "admin" || password != "secret" {
    return echo.NewHTTPError(http.StatusUnauthorized, "Invalid credentials")
}
// 验证通过，继续执行后续逻辑
return next(c)
```

#### **（2）上下文传递**
验证通过后，可将用户信息存入上下文供后续使用：
```go
c.Set("user", &User{Name: username, Role: "admin"})
return next(c)
```

---

### **4. 与标准中间件的对比**
- **与`middleware.BasicAuth`的区别**：  
  Echo内置的`BasicAuth`中间件已封装了凭证提取和验证流程，开发者只需提供验证函数。而本例是自定义中间件的简化版，需手动实现完整逻辑。
- **性能影响**：  
  若在中间件中执行数据库查询等耗时操作，可能成为性能瓶颈。建议结合缓存或JWT等无状态验证方式优化。

---

### **5. 注意事项**
1. **中间件顺序**：  
   全局中间件按注册顺序执行，若需依赖其他中间件（如日志记录），应调整注册顺序。
2. **错误处理**：  
   可通过Echo的`HTTPErrorHandler`自定义错误响应格式。
3. **安全加固**：  
   实际场景中需防范暴力破解，可结合速率限制中间件（如`middleware.RateLimiter`）。

---

### **总结**
这段代码演示了Echo中间件的**基础拦截机制**，但缺乏实际认证逻辑。完整实现需补充凭证提取、验证及上下文传递功能，其设计模式体现了责任链模式的思想——通过`next(c)`控制请求流转。在实际开发中，建议优先使用框架内置的认证中间件，仅在需要深度定制时采用此自定义方案。


# 默认 HTTP 错误处理程序

Echo 提供了默认的 HTTP 错误处理程序，它用 JSON 格式发送错误。

```go
{
  "message": "error connecting to redis"
}
```

标准错误 `error` 的响应是 `500 - Internal Server Error`。然而在调试 (debug) 模式模式下，原始的错误信息会被发送。如果错误是 `*HTTPError`，则使用设置的状态代码和消息发送响应。如果启用了日志记录，则还会记录错误消息。


# 自定义HTTP错误处理程序

通过 `e.HTTPErrorHandler` 可以设置自定义的 HTTP 错误处理程序 (error handler) 。

通常默认的 HTTP 错误处理程序已经够用；然而如果要获取不同类型的错误并采取相应的操作，则可以使用自定义 HTTP 错误处理程序，例如发送通知邮件或记录日志到应用中心的场景。最后，你还可以发送自定义的错误页面或 JSON 响应给客户端。


## 错误页

利用自定义 HTTP 错误处理程序，可以在显示不同种类的错误页面的同时，记录错误日志。错误页的名称可写作 `<CODE>.html`，例如 `500.html`。你可以在https://github.com/AndiDittrich/HttpErrorPages看到 Echo 内置的错误页。

```go
func customHTTPErrorHandler(err error, c echo.Context) {
    code := http.StatusInternalServerError
    if he, ok := err.(*echo.HTTPError); ok {
        code = he.Code
    }
    errorPage := fmt.Sprintf("%d.html", code)
    if err := c.File(errorPage); err != nil {
        c.Logger().Error(err)
    }
    c.Logger().Error(err)
}

e.HTTPErrorHandler = customHTTPErrorHandler
```


`日志除了记录到 logger，也可以记录到第三方服务，例如 Elasticsearch 或者 Splunk。`


---

这段代码是 **Echo框架中自定义HTTP错误处理器的实现**，主要用于统一处理Web应用中的各类错误，并返回对应的错误页面。以下是关键步骤的详细解析：

---

### **1. 错误处理逻辑拆解**
```go
func customHTTPErrorHandler(err error, c echo.Context) {
    // 默认状态码为500（服务器内部错误）
    code := http.StatusInternalServerError
    // 尝试将错误转换为Echo的HTTPError类型
    if he, ok := err.(*echo.HTTPError); ok {
        code = he.Code // 提取具体HTTP状态码（如404、401等）
    }
    // 根据状态码生成对应的HTML文件名（如404.html）
    errorPage := fmt.Sprintf("%d.html", code)
    // 尝试发送错误页面文件
    if err := c.File(errorPage); err != nil {
        // 若文件不存在，记录日志
        c.Logger().Error(err)
    }
    // 无论是否成功返回页面，记录原始错误日志
    c.Logger().Error(err)
}
```
- **核心功能**：
    - **动态匹配错误页面**：根据HTTP状态码（如404、500）自动查找对应的`状态码.html`文件（如`404.html`）并返回。
    - **兼容多种错误类型**：支持处理标准`error`和Echo内置的`HTTPError`。
    - **统一日志记录**：无论是否找到错误页面，均记录原始错误信息到日志。

---

### **2. 注册自定义错误处理器**
```go
e.HTTPErrorHandler = customHTTPErrorHandler
```
- **作用**：将上述自定义函数设为Echo实例的全局错误处理器，覆盖框架默认的JSON错误响应。
- **触发场景**：
    - 路由未找到（404）
    - 中间件认证失败（如401）
    - 控制器返回错误（如`return echo.NewHTTPError(500, "数据库错误")`）

---

### **3. 对比默认行为**
- **默认错误处理**：
    - 返回JSON格式错误（如`{"message":"Not Found"}`）。
    - 仅记录错误日志，不返回自定义页面。
- **自定义优势**：
    - **用户体验优化**：通过友好的HTML页面替代原始错误信息。
    - **灵活扩展**：可为不同错误类型设计不同页面（如`401.html`用于认证失败）。
    - **运维友好**：日志中保留错误细节，便于调试。

---

### **4. 潜在问题与改进建议**
- **文件缺失风险**：
    - 若`404.html`等文件不存在，会触发`c.File`的二次错误，需添加回退机制：
      ```go
      if err := c.File(errorPage); err != nil {
          // 返回默认错误页或纯文本
          c.String(code, "Error %d: %s", code, err.Error())
      }
      ```
- **状态码覆盖**：
    - 非`HTTPError`类型的错误（如普通`error`）会被统一视为500错误，需根据业务需求扩展判断逻辑。

---

### **5. 实际应用场景**
- **REST API开发**：结合`c.JSON`返回结构化错误信息（需修改代码逻辑）。
- **企业级后台系统**：为不同部门定制错误页（如财务系统使用`/errors/401_finance.html`）。
- **高安全场景**：在401错误页中添加重新登录引导或双因素认证入口。

---

### **总结**
这段代码通过自定义错误处理器，将Echo框架的错误响应从默认的JSON格式升级为动态HTML页面，同时保留详细的日志记录功能。其核心价值在于**统一错误处理流程**与**提升终端用户友好性**，是构建高可用Web应用的常见实践。