# 快速开始

## 安装

```bash
go get github.com/labstack/echo/v4
```

## 编写Hello,World!

```go
package main

import (
	"github.com/labstack/echo/v4"
	"net/http"
)

func main() {
	e := echo.New()
	e.GET("/", func(c echo.Context) error {
		return c.String(http.StatusOK, "Hello, World!")
	})
	e.Logger.Fatal(e.Start(":1323"))
}

```

## 路由

```go
e.POST("/users", saveUser)
e.GET("/users/:id", getUser)
e.PUT("/users/:id", updateUser)
e.DELETE("/users/:id", deleteUser)
```

## URL路径参数

```go
// e.GET("/users/:id", getUser)
func getUser(c echo.Context) error {
    // User ID 来自于url `users/:id`
    id := c.Param("id")
    return c.String(http.StatusOK, id)
}
```

用浏览器访问 http://localhost:1323/users/Joe 然后就能在页面上看到 `Joe`

## 请求参数

`/show?team=x-men&member=wolverine`


```go
// e.GET("/show", show)
func show(c echo.Context) error {
    // 从请求参数里获取 team 和 member 的值
    team := c.QueryParam("team")
    member := c.QueryParam("member")
    return c.String(http.StatusOK, "team:" + team + ", member:" + member)
}
```

从浏览器访问 `http://localhost:1323/show?team=x-men&member=wolverine` 可以看到页面上显示”team:x-men, member:wolverine”


## 表单 `application/x-www-form-urlencoded`

`POST` `/save`

| name  | value            |
|-------|------------------|
| name  | Joe Smith        |
| email | joe@labstack.com |

```go
// e.POST("/save", save)
func save(c echo.Context) error {
    // 获取 name 和 email 的值
    name := c.FormValue("name")
    email := c.FormValue("email")
    return c.String(http.StatusOK, "name:" + name + ", email:" + email)
}
```

在命令行里执行下面的语句

```go
$ curl -F "name=Joe Smith" -F "email=joe@labstack.com" http://localhost:1323/save
```

控制台会输出`name:Joe Smith, email:joe@labstack.com`


# 表单 `multipart/form-data`

`POST` `/save`

| name   | value    |
|--------|----------|
| name   | Joe Smith |
| avatar | avatar   |

```go
func save(c echo.Context) error {
    // Get name
    name := c.FormValue("name")
    // Get avatar
      avatar, err := c.FormFile("avatar")
      if err != nil {
         return err
     }

     // Source
     src, err := avatar.Open()
     if err != nil {
         return err
     }
     defer src.Close()

     // Destination
     dst, err := os.Create(avatar.Filename)
     if err != nil {
         return err
     }
     defer dst.Close()

     // Copy
     if _, err = io.Copy(dst, src); err != nil {
          return err
      }

    return c.HTML(http.StatusOK, "<b>Thank you! " + name + "</b>")
}
```


命令行执行下面语句

```go
$ curl -F "name=Joe Smith" -F "avatar=@/path/to/your/avatar.png" http://localhost:1323/save
//output => <b>Thank you! Joe Smith</b>
```

使用以下命令查看刚刚上传的图片

```go
cd <project directory>
ls avatar.png
// => avatar.png
```

# 处理请求

- 根据 Content-Type 请求标头将 `json`，`xml`，`form` 或 `query` 负载绑定到 Go 结构中。
- 通过状态码将响应渲染为 `json` 或者 `xml` 格式。

```go
type User struct {
    Name  string `json:"name" xml:"name" form:"name" query:"name"`
    Email string `json:"email" xml:"email" form:"email" query:"email"`
}

e.POST("/users", func(c echo.Context) error {
    u := new(User)
    if err := c.Bind(u); err != nil {
        return err
    }
    return c.JSON(http.StatusCreated, u)
    // 或者
    // return c.XML(http.StatusCreated, u)
})
```

# 静态资源

下面的代码定义`/static/*`目录为静态资源文件目录

```go
e.Static("/static", "static")
```


# 中间件

```go
// Root level middleware
e.Use(middleware.Logger())
e.Use(middleware.Recover())

// Group level middleware
g := e.Group("/admin")
g.Use(middleware.BasicAuth(func(username, password string, c echo.Context) (error, bool) {
  if username == "joe" && password == "secret" {
    return nil, true
  }
  return nil, false
}))

// Route level middleware
track := func(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        println("request to /users")
        return next(c)
    }
}
e.GET("/users", func(c echo.Context) error {
    return c.String(http.StatusOK, "/users")
}, track)
```

这段代码展示了Go语言Echo框架中**中间件的分层应用**，具体分为根级、组级和路由级三种中间件的注册方式。以下是对各部分代码的详细解释：

---

### **1. 根级中间件（全局中间件）**
```go
e.Use(middleware.Logger())
e.Use(middleware.Recover())
```
- **作用**：这两行代码注册了全局中间件，对所有路由生效。
    - `Logger()`：记录HTTP请求的日志（如请求方法、路径、状态码等）。
    - `Recover()`：捕获路由处理过程中的panic，防止服务崩溃。
- **执行顺序**：全局中间件按注册顺序执行，因此所有请求会先经过`Logger`，再经过`Recover`。

---

### **2. 组级中间件（分组路由中间件）**
```go
g := e.Group("/admin")
g.Use(middleware.BasicAuth(func(username, password string, c echo.Context) (error, bool) {
  if username == "joe" && password == "secret" {
    return nil, true  // 认证通过
  }
  return nil, false  // 认证失败
}))
```
- **作用**：为`/admin`路径下的所有路由添加中间件。
    - `BasicAuth`：实现HTTP基本认证，只有用户名和密码匹配时（此处为`joe/secret`），才允许访问后续路由。
- **特性**：组级中间件仅作用于该分组内的路由。例如，访问`/admin/*`时，会先执行全局中间件（Logger、Recover），再执行组级中间件（BasicAuth）。

---

### **3. 路由级中间件（特定路由中间件）**
```go
track := func(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        println("request to /users")
        return next(c)
    }
}
e.GET("/users", func(c echo.Context) error {
    return c.String(http.StatusOK, "/users")
}, track)
```
- **作用**：为`/users`路由单独添加中间件`track`。
    - `track`：在请求到达路由处理函数前打印日志（`request to /users`），然后通过`next(c)`将控制权交给下一个中间件或路由处理器。
- **特性**：路由级中间件仅作用于该路由，执行顺序为全局中间件 → 路由级中间件 → 路由处理函数。

---

### **中间件执行流程总结**
以访问`/users`为例，执行顺序为：
1. **全局中间件**：`Logger` → `Recover`
2. **路由级中间件**：`track`
3. **路由处理函数**：返回响应内容`/users`

以访问`/admin`下的路由（如`/admin/dashboard`）为例，执行顺序为：
1. **全局中间件**：`Logger` → `Recover`
2. **组级中间件**：`BasicAuth`
3. **路由处理函数**（假设存在）

---

### **关键机制解析**
- **中间件链式调用**：Echo中间件通过`next(c)`将请求传递给下一个中间件或路由处理器，形成链式调用。
- **中间件作用域**：支持全局、分组、路由三个层级，灵活控制中间件的生效范围。
- **自定义逻辑**：中间件可在请求前后插入逻辑（如日志、认证、性能监控等），增强请求处理能力。

通过这种分层设计，Echo框架实现了中间件的灵活组合与高效管理。