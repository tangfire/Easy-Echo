# 请求

## 数据绑定


可以使用 `Context#Bind(i interface{})` 将请求内容体绑定至 go 的结构体。默认绑定器支持基于 Content-Type 标头包含 application/json，application/xml 和 application/x-www-form-urlencoded 的数据。

下面是绑定请求数据到 `User` 结构体的例子


```go
// User
User struct {
  Name  string `json:"name" form:"name" query:"name"`
  Email string `json:"email" form:"email" query:"email"`
}
```


```go
// Handler
func(c echo.Context) (err error) {
  u := new(User)
  if err = c.Bind(u); err != nil {
    return
  }
  return c.JSON(http.StatusOK, u)
}
```

## JSON数据

```go
curl \
  -X POST \
  http://localhost:1323/users \
  -H 'Content-Type: application/json' \
  -d '{"name":"Joe","email":"joe@labstack"}'
```


## Form表单数据

```go
curl \
  -X POST \
  http://localhost:1323/users \
  -d 'name=Joe' \
  -d 'email=joe@labstack.com'
```

## 查询参数 (Query Parameters)

```go
curl \
  -X GET \
  http://localhost:1323/users\?name\=Joe\&email\=joe@labstack.com
```

# 自定义绑定器

可以通过 `Echo#Binder` 注册自定义绑定器


示例


```go
type CustomBinder struct {}

func (cb *CustomBinder) Bind(i interface{}, c echo.Context) (err error) {
    // 你也许会用到默认的绑定器
    db := new(echo.DefaultBinder)
    if err = db.Bind(i, c); err != echo.ErrUnsupportedMediaType {
        return
    }

    // 做你自己的实现

    return
}
```


这段代码实现了一个基于Go语言Echo框架的自定义请求数据绑定器（`CustomBinder`），其核心逻辑是**扩展默认绑定功能**，并允许开发者针对特定需求添加自定义绑定逻辑。以下是逐层解析：

---

### 1. **代码结构解析**
```go
type CustomBinder struct {}  // 定义空结构体作为自定义绑定器的载体

func (cb *CustomBinder) Bind(i interface{}, c echo.Context) (err error) {
    // 调用默认绑定器
    db := new(echo.DefaultBinder)
    if err = db.Bind(i, c); err != echo.ErrUnsupportedMediaType {
        return
    }

    // 自定义绑定逻辑（此处为示例占位符）
    
    return
}
```

---

### 2. **关键逻辑说明**
#### （1）**默认绑定器的复用**
- **目的**：通过`echo.DefaultBinder`处理常见请求类型（如`JSON`/`Form`/`Query`等）。
- **错误处理**：若默认绑定器返回的错误**不是**`ErrUnsupportedMediaType`（如参数类型不匹配），则直接返回该错误，**不执行后续自定义逻辑**。这表示默认绑定器已处理了可识别的请求类型，无需重复处理。

#### （2）**自定义逻辑的触发条件**
- **场景**：当请求的媒体类型（如`Content-Type`）**不被默认绑定器支持**时（即`err == echo.ErrUnsupportedMediaType`），触发自定义绑定逻辑。
- **应用示例**：可在此处解析如`XML`、`Protobuf`等非标准格式，或实现特殊数据验证（如加密参数解密）。

---

### 3. **设计模式与最佳实践**
- **扩展性**：通过组合默认绑定器，避免了重复造轮子，仅补充框架未覆盖的绑定场景。
- **接口兼容性**：`Bind`方法需符合Echo的`Binder`接口定义（绑定数据到`i`并返回错误），确保与框架无缝集成。
- **指针接收者**：使用`(cb *CustomBinder)`作为方法接收者，允许在必要时维护绑定器的内部状态（如缓存、配置等）。

---

### 4. **实际应用示例**
假设需要解析自定义的`text/plain`请求体为结构化数据，可在`// 做你自己的实现`部分添加以下逻辑：
```go
// 示例：解析纯文本格式的请求体
contentType := c.Request().Header.Get("Content-Type")
if contentType == "text/plain" {
    body, _ := io.ReadAll(c.Request().Body)
    // 将文本内容解析到目标结构体i
    if err := customTextParser(i, body); err != nil {
        return err
    }
    return nil
}
```

---

### 5. **与其他语言实现的对比**
- **与C#模型绑定的异同**：
    - **相似点**：均通过继承/实现接口（如C#的`IModelBinder`）扩展绑定逻辑。
    - **差异点**：Go的`CustomBinder`更强调对默认行为的复用，而C#通常需显式注册绑定器到依赖注入容器。

---

### 总结
这段代码提供了一种**渐进式增强**的绑定策略：优先依赖Echo框架的默认行为，仅在必要时介入自定义逻辑。这种设计既保证了通用场景的效率，又为特殊需求保留了灵活性，是中间件开发中的典型实践。


# 检索数据

## Form表单数据

表单数据可以通过名称检索，使用`Context#FormValue(name string)`方法。

示例

```go
// Handler
func(c echo.Context) error {
    name := c.FormValue("name")
    return c.String(http.StatusOK, name)
}
```

```go
curl \
  -X POST \
  http://localhost:1323 \
  -d 'name=Joe'
```

当然，你也可以通过实现 `Echo#BindUnmarshaler` `接口来绑定自定义数据类型。

```go
type Timestamp time.Time

func (t *Timestamp) UnmarshalParam(src string) error {
    ts, err := time.Parse(time.RFC3339, src)
    *t = Timestamp(ts)
    return err
}
```

## 查询参数(Query Parameters)

查询参数可以通过名称获取，使用 `Context#QueryParam(name string)` 方法。

示例


```go
// Handler
func(c echo.Context) error {
    name := c.QueryParam("name")
    return c.String(http.StatusOK, name)
})
```

```go
curl \
  -X GET \
  http://localhost:1323\?name\=Joe
```

和表单数据一样，自定义数据类型依然通过 `Context#QueryParam(name string)` 进行绑定。


## 路径参数 (Path Parameters)

路径参数可以通过 `Context#Param(name string) string` 进行检索。

示例

```go
e.GET("/users/:name", func(c echo.Context) error {
    name := c.Param("name")
    return c.String(http.StatusOK, name)
})

```

```go
$ curl http://localhost:1323/users/Joe
```

## 数据验证

Echo 没有内置的数据验证功能，但是可以通过 Echo#Validator 和第三方库来注册一个数据验证器。

下面例子使用 https://github.com/go-playground/validator 所展示的框架来做验证：


```go
type (
    User struct {
        Name  string `json:"name" validate:"required"`
        Email string `json:"email" validate:"required,email"`
    }

    CustomValidator struct {
        validator *validator.Validate
    }
)

func (cv *CustomValidator) Validate(i interface{}) error {
    return cv.validator.Struct(i)
}

func main() {
    e := echo.New()
    e.Validator = &CustomValidator{validator: validator.New()}
    e.POST("/users", func(c echo.Context) (err error) {
        u := new(User)
        if err = c.Bind(u); err != nil {
            return
        }
        if err = c.Validate(u); err != nil {
            return
        }
        return c.JSON(http.StatusOK, u)
    })
    e.Logger.Fatal(e.Start(":1323"))
}
```


这段代码是一个使用Go语言Echo框架实现的HTTP服务，核心功能是接收用户数据并进行验证。以下是逐层解析：

---

### 一、结构体定义与验证标签
```go
type User struct {
    Name  string `json:"name" validate:"required"`
    Email string `json:"email" validate:"required,email"`
}
```
- **`json`标签**：定义JSON序列化/反序列化时的字段名称。例如，请求体的`{"name": "Alice"}`会被映射到`User.Name`字段。
- **`validate`标签**：指定验证规则：
  - `required`：字段不能为空。
  - `email`：必须符合邮箱格式（依赖第三方验证库如`go-playground/validator`）。

---

### 二、自定义验证器实现
```go
type CustomValidator struct {
    validator *validator.Validate
}

func (cv *CustomValidator) Validate(i interface{}) error {
    return cv.validator.Struct(i)
}
```
- **作用**：将第三方验证库（如`validator`）适配到Echo框架的验证接口。Echo要求验证器实现`Validate(interface{}) error`方法。
- **实现逻辑**：调用`validator.Struct(i)`会根据结构体的`validate`标签自动校验字段。

---

### 三、主函数逻辑
```go
func main() {
    e := echo.New()
    e.Validator = &CustomValidator{validator: validator.New()}
    // 注册路由和处理函数...
}
```
1. **初始化Echo实例**：创建HTTP服务器基础环境。
2. **注入自定义验证器**：使Echo在调用`c.Validate()`时使用第三方库进行校验。

---

### 四、路由与请求处理
```go
e.POST("/users", func(c echo.Context) error {
    u := new(User)
    if err := c.Bind(u); err != nil { /* 解析请求体到结构体 */ }
    if err := c.Validate(u); err != nil { /* 触发验证 */ }
    return c.JSON(http.StatusOK, u)
})
```
- **`c.Bind(u)`**：将请求体（如JSON）自动解析到`User`结构体，依赖`json`标签。
- **`c.Validate(u)`**：调用自定义验证器，检查`validate`标签规则：
  - 若`Name`为空或`Email`格式错误，返回HTTP 400错误。
- **成功响应**：返回HTTP 200及用户数据。

---

### 五、技术整合分析
1. **Echo框架角色**：负责路由管理、中间件集成和请求上下文处理。
2. **验证库协作**：通过结构体标签声明规则，验证器动态解析标签并执行校验。
3. **错误处理**：验证失败时，错误信息会通过Echo自动转换为JSON响应。

---

### 六、运行示例
1. **合法请求**：
   ```bash
   curl -X POST -H "Content-Type: application/json" -d '{"name":"Alice","email":"alice@example.com"}' http://localhost:1323/users
   ```
  - 响应：`200 OK`及用户数据。
2. **非法请求**（如缺少`name`字段）：
  - 响应：`400 Bad Request`及错误详情（如`Key: 'User.Name' Error:Field validation for 'Name' failed on the 'required' tag`）。

---

通过这种方式，代码实现了高效的数据验证与HTTP服务逻辑的解耦，符合Go语言“组合优于继承”的设计哲学。


```go
curl \
  -X POST \
  http://localhost:1323/users \
  -H 'Content-Type: application/json' \
  -d '{"name":"Joe","email":"joe@invalid-domain"}'
{"message":"Key: 'User.Email' Error:Field validation for 'Email' failed on the 'email' tag"}

```

