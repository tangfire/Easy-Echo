# Context

echo.Context 表示当前 HTTP 请求的上下文。通过路径、路径参数、数据、注册处理器和相关 API 进行请求的读取与响应的输出。由于 Context 是一个接口，也可以轻松地使用自定义 API 进行扩展。

# 扩展Context

## 定义一个自定义 context

```go
type CustomContext struct {
    echo.Context
}

func (c *CustomContext) Foo() {
    println("foo")
}

func (c *CustomContext) Bar() {
    println("bar")
}
```

创建一个中间件来扩展默认的 context

```go
e.Use(func(h echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        cc := &CustomContext{c}
        return h(cc)
    }
})
```

`此中间件应在任何其他中间件之前注册。`

在处理器中使用

```go
e.Get("/", func(c echo.Context) error {
    cc := c.(*CustomContext)
    cc.Foo()
    cc.Bar()
    return cc.String(200, "OK")
})
```



---

这段代码是 **Echo框架中自定义上下文（Custom Context）的中间件实现**，其核心作用是将标准的`echo.Context`替换为开发者扩展的自定义上下文对象，从而在全局范围内增强请求处理能力。以下是详细解析：

---

### **1. 代码结构解析**
```go
e.Use(func(h echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        cc := &CustomContext{c}  // 创建自定义上下文
        return h(cc)             // 将自定义上下文传递给后续中间件/控制器
    }
})
```
- **功能逻辑**：
    - 通过`e.Use()`注册一个全局中间件。
    - 中间件接收原始的`echo.HandlerFunc`（即后续中间件或路由处理函数），返回一个新的`HandlerFunc`。
    - 当请求到达时，新`HandlerFunc`会将标准的`echo.Context`包装到自定义的`CustomContext`中，再调用原始`h`函数，并将自定义上下文`cc`传递下去。

---

### **2. 关键机制**
#### **（1）自定义上下文的作用**
- **扩展功能**：`CustomContext`通常继承自`echo.Context`，开发者可以添加自定义方法或属性（如数据库连接、用户认证信息等）。
  ```go
  type CustomContext struct {
      echo.Context
      // 添加自定义字段或方法，例如：
      User *User
  }
  ```
- **全局共享**：所有后续中间件和路由处理函数中，通过`c.Get("...")`或类型断言`c.(*CustomContext)`可访问自定义字段。

#### **（2）中间件的链式调用**
- **执行顺序**：中间件通过`h(cc)`将控制权交给下一个中间件或路由处理器，形成链式调用。
- **上下文传递**：自定义上下文会贯穿整个请求生命周期，确保后续逻辑统一使用增强后的上下文对象。

---

### **3. 实际应用场景**
#### **（1）存储全局数据**
- 例如在中间件中解析JWT令牌，将用户信息存入自定义上下文，供后续控制器直接使用：
  ```go
  func AuthMiddleware(next echo.HandlerFunc) echo.HandlerFunc {
      return func(c echo.Context) error {
          token := parseToken(c.Request())
          cc := c.(*CustomContext)
          cc.User = token.User  // 存储用户信息
          return next(cc)
      }
  }
  ```

#### **（2）增强请求处理能力**
- 添加自定义方法如日志记录、性能监控等：
  ```go
  func (cc *CustomContext) Log(message string) {
      fmt.Printf("[%s] %s\n", time.Now().Format(time.RFC3339), message)
  }
  ```

#### **（3）统一处理逻辑**
- 在自定义上下文中封装重复操作（如参数校验、响应格式化），减少控制器代码冗余。

---

### **4. 与其他中间件的对比**
- **与参数传递中间件的区别**：  
  普通中间件通过`c.Set()`/`c.Get()`传递简单数据，而自定义上下文可直接绑定结构化数据和逻辑方法，更适用于复杂场景。
- **与框架内置中间件的关系**：  
  此中间件运行在全局层级，优先于其他中间件（如Logger、Recover），确保自定义上下文在后续流程中生效。

---

### **5. 注意事项**
1. **类型断言安全**：在控制器中使用自定义上下文时，需通过类型断言确保上下文类型正确：
   ```go
   func MyController(c echo.Context) error {
       cc, ok := c.(*CustomContext)
       if !ok {
           return errors.New("invalid context type")
       }
       cc.Log("Request received")
       // ...
   }
   ```
2. **中间件顺序**：若需依赖其他中间件（如Logger），应在其之后注册自定义上下文中间件，避免上下文未初始化的问题。

---

### **总结**
这段代码通过全局中间件将Echo框架的默认上下文替换为自定义上下文，实现了**功能扩展**与**数据共享**的深度融合。这种模式在需要全局增强请求处理能力的场景（如用户身份绑定、统一日志记录）中极为实用，是Echo框架高阶用法的典型实践。
