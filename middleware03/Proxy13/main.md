# Proxy (代理) 中间件

Proxy 提供 HTTP / WebSocket 反向代理中间件。它使用已配置的负载平衡技术将请求转发到上游服务器。

用法


```go
url1, err := url.Parse("http://localhost:8081")
if err != nil {
  e.Logger.Fatal(err)
}
url2, err := url.Parse("http://localhost:8082")
if err != nil {
  e.Logger.Fatal(err)
}
e.Use(middleware.Proxy(&middleware.RoundRobinBalancer{
  Targets: []*middleware.ProxyTarget{
    {
      URL: url1,
    },
    {
      URL: url2,
    },
  },
}))
```

# 自定义配置

用法

```go
e := echo.New()
e.Use(middleware.ProxyWithConfig(middleware.ProxyConfig{}))
```

# 配置

```go
// ProxyConfig defines the config for Proxy middleware.
ProxyConfig struct {
// Skipper defines a function to skip middleware.
Skipper Skipper

// Balancer defines a load balancing technique.
// Required.
// Possible values:
// - RandomBalancer
// - RoundRobinBalancer
Balancer ProxyBalancer
}
```

这段代码用于在 Go 的 Web 框架（如 Echo 框架）中配置一个反向代理中间件，通过 **轮询（Round Robin）负载均衡策略** 将请求分发到两个本地后端服务（分别运行在 8081 和 8082 端口）。以下是分步解析：

---

### **1. 解析后端服务的 URL**
```go
url1, err := url.Parse("http://localhost:8081")
if err != nil {
  e.Logger.Fatal(err)
}
url2, err := url.Parse("http://localhost:8082")
if err != nil {
  e.Logger.Fatal(err)
}
```
- **目的**：将字符串格式的 URL 解析为 Go 的 `*url.URL` 对象，供代理中间件使用。
- **`url.Parse`**：检查 URL 格式是否合法（例如协议、主机名、端口是否正确）。
- **错误处理**：如果解析失败（如 URL 格式错误），直接记录错误并终止程序（`Fatal` 会退出应用）。

---

### **2. 配置反向代理与负载均衡**
```go
e.Use(middleware.Proxy(&middleware.RoundRobinBalancer{
  Targets: []*middleware.ProxyTarget{
    { URL: url1 },
    { URL: url2 },
  },
}))
```
- **`e.Use`**：向 Web 框架（例如 Echo）注册中间件，所有请求都会先经过这个中间件处理。
- **`middleware.Proxy`**：代理中间件，负责将请求转发到指定的后端服务。
- **`RoundRobinBalancer`**：负载均衡器，按顺序轮流将请求分发到 `Targets` 列表中的目标。
    - 例如：第一个请求发给 `8081`，第二个发给 `8082`，第三个又回到 `8081`，依此类推。
- **`Targets`**：定义了两个后端服务地址（`url1` 和 `url2`），即代理的目标。

---

### **3. 工作流程**
1. **接收请求**：当客户端发送请求到当前服务（例如 `http://localhost:80`），代理中间件会拦截请求。
2. **选择目标**：负载均衡器（`RoundRobinBalancer`）按顺序选择一个后端服务（如 `8081` 或 `8082`）。
3. **转发请求**：将客户端的请求转发到选中的后端服务，并将响应返回给客户端。

---

### **核心作用**
- **反向代理**：隐藏后端服务细节，客户端只需与代理服务交互。
- **负载均衡**：分散请求到多个后端实例，避免单点过载，提升系统吞吐量和可用性。
- **高可用性**：如果某个后端服务宕机，负载均衡器会自动跳过故障实例（需中间件支持健康检查，但此代码未显式配置）。

---

### **适用场景**
- 后端服务需要横向扩展（例如部署多个实例处理高并发）。
- 需要灰度发布或 A/B 测试时，将流量分发到不同版本的服务。

---

### **补充说明**
- **框架推测**：代码中的 `e` 可能是 Echo 框架的实例，`middleware.Proxy` 是 Echo 的中间件。
- **扩展性**：可添加更多 `Targets`（例如 `url3`, `url4`）以支持更多后端实例。
- **其他负载均衡策略**：除了轮询，还可能支持加权轮询、最少连接数等（需中间件支持）。

如果需要调试或扩展此代码，可以添加日志、超时设置或健康检查逻辑。


默认配置

| 名称      | 值              |
|---------|----------------|
| Skipper | DefaultSkipper |



