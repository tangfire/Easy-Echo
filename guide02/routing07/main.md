# 路由

基于 radix tree ，Echo 的路由查询速度非常快。路由使用 sync pool 来重用内存，实现无 GC 开销下的零动态内存分配。

通过特定的 HTTP 方法，url 路径和一个匹配的处理程序 (handler) 可以注册一个路由。例如，下面的代码则展示了一个注册路由的例子：它包括 `Get` 的访问方式， `/hello` 的访问路径，以及发送 `Hello World` HTTP 响应的处理程序。

```go
// 业务处理
func hello(c echo.Context) error {
      return c.String(http.StatusOK, "Hello, World!")
}

// 路由
e.GET("/hello", hello)
```


你可以用 `Echo.Any(path string, h Handler)` 来为所有的 HTTP 方法发送注册 处理程序(handler)；如果仅需要为某个特定的方法注册路由，可使用 `Echo.Match(methods []string, path string, h Handler)`。

Echo 通过 `func(echo.Context) error` 定义 handler 方法，其中 `echo.Context` 已经内嵌了 HTTP 请求和响应接口。


## 匹配所有

匹配零个或多个字符的路径。例如， `/users/*` 将会匹配；

- `/users/`
- `/users/1`
- `/users/1/files/1`
- `/users/anything...`



## 路径匹配顺序

- Static (固定路径)
- Param (参数路径)
- Match any (匹配所有)

实例

```go
e.GET("/users/:id", func(c echo.Context) error {
    return c.String(http.StatusOK, "/users/:id")
})

e.GET("/users/new", func(c echo.Context) error {
    return c.String(http.StatusOK, "/users/new")
})

e.GET("/users/1/files/*", func(c echo.Context) error {
    return c.String(http.StatusOK, "/users/1/files/*")
})
```

上面定义的路由将按下面的优先级顺序匹配:

- `/users/new`
- `/users/:id`
- `/users/1/files/*`

路由可以按照任意顺序定义。


## 组路由

`Echo#Group(prefix string, m ...Middleware) *Group`

可以将具有相同前缀的路由归为一组从而定义具有可选中间件的新子路由。除了一些特殊的中间件外，组路由也会继承父中间件。若要在组路由中添加中间件，则需使用 `Group.Use(m ...Middleware)` 。最后，组路由也可以嵌套。

下面的代码，我们创建了一个 admin 组，它需要对 `/admin/*` 路由进行基本的 HTTP 身份认证。


```go
g := e.Group("/admin")
g.Use(middleware.BasicAuth(func(username, password string) bool {
    if username == "joe" && password == "secret" {
        return true
    }
    return false
}))
```

## 路由命名

每个路由都会返回一个 `Route` 对象，这个对象可以用来给路由命名。比如：

```go
routeInfo := e.GET("/users/:id", func(c echo.Context) error {
    return c.String(http.StatusOK, "/users/:id")
})
routeInfo.Name = "user"

// 或者这样写
e.GET("/users/new", func(c echo.Context) error {
    return c.String(http.StatusOK, "/users/new")
}).Name = "newuser"
```

当你需要在模版生成 uri 但是又无法获取路由的引用，或者多个路由使用相同的处理器(handler)的时候，路由命名就会显得更方便。

## 构造URI

`Echo#URI(handler HandlerFunc, params ...interface{})` 可以用来在任何业务处理代码里生成带有特殊参数的URI。这样在你重构自己的应用程序的时候，可以很方便的集中处理所有的 URI 。

除了 `Echo#URI`，还可以使用 `Echo#Reverse(name string, params ...interface{})` 方法根据路由名生成 uri。比如，当 `foobar` 进行如下设置时，使用 `Echo#Reverse("foobar", 1234)` 就会生成 `/users/1234` ：


以下为基于 Echo 框架的 `URI()` 和 `Reverse()` 方法代码实现示例，结合用户提供的上下文及文档规范：

---

### **1. 使用 `Echo#URI` 生成带参数的 URI**
```go
package main

import (
    "github.com/labstack/echo/v4"
    "net/http"
)

// 定义处理函数
func getUserHandler(c echo.Context) error {
    id := c.Param("id")
    return c.String(http.StatusOK, "User ID: "+id)
}

func main() {
    e := echo.New()
    
    // 注册路由（路径参数为 :id）
    e.GET("/users/:id", getUserHandler)
    
    // 在业务代码中生成 URI
    generatedURI := e.URI(getUserHandler, 1)
    println("Generated URI:", generatedURI) // 输出 /users/1
    
    e.Start(":8080")
}
```
**说明**：
- `e.URI()` 的第一个参数是路由绑定的处理函数 `HandlerFunc`，后续参数对应路径中的动态参数（如 `:id` 的值 `1`）。
- 此方法适用于通过函数引用直接生成 URI，无需依赖路由名。

---

### **2. 使用 `Echo#Reverse` 通过路由名生成 URI**
```go
package main

import (
    "github.com/labstack/echo/v4"
    "net/http"
)

func getUserHandler(c echo.Context) error {
    id := c.Param("id")
    return c.String(http.StatusOK, "User ID: "+id)
}

func main() {
    e := echo.New()
    
    // 注册路由并命名（路由名为 "foobar"）
    e.GET("/users/:id", getUserHandler).Name = "foobar"
    
    // 通过路由名和参数生成 URI
    generatedURI := e.Reverse("foobar", 1234)
    println("Generated URI:", generatedURI) // 输出 /users/1234
    
    e.Start(":8080")
}
```
**说明**：
- 通过 `.Name` 属性为路由命名（如 `"foobar"`），之后可通过 `e.Reverse()` 根据名称和参数生成 URI。
- 此方法在重构时更灵活，尤其适用于多个路由共用同一处理函数或需要集中管理 URI 的场景。

---

### **核心区别**
| 方法          | 依赖对象       | 适用场景                         | 参数传递方式       |
|---------------|---------------|--------------------------------|-------------------|
| `Echo#URI()`  | 处理函数引用    | 直接关联函数逻辑的 URI 生成       | 动态参数直接传入    |
| `Echo#Reverse()`| 路由名称       | 通过名称解耦，便于集中管理路由配置 | 参数需按顺序匹配   |

**应用建议**：
- 若需要直接关联处理函数逻辑，优先使用 `URI()`；
- 若需解耦路由名称与处理函数（如动态生成链接或 API 文档），优先使用 `Reverse()`。

完整路由配置示例可参考 Echo 官方文档。


## 路由列表


`Echo#Routes() []*Route` 会根据路由定义的顺序列出所有已经注册的路由。每一个路由包含 http 方法，路径和对应的处理器(handler)。

示例

```go
// Handlers
func createUser(c echo.Context) error {
}

func findUser(c echo.Context) error {
}

func updateUser(c echo.Context) error {
}

func deleteUser(c echo.Context) error {
}

// Routes
e.POST("/users", createUser)
e.GET("/users", findUser)
e.PUT("/users", updateUser)
e.DELETE("/users", deleteUser)

```
用下面的代码你将所有的路由信息输出到 JSON 文件：


```go
data, err := json.MarshalIndent(e.Routes(), "", "  ")
if err != nil {
    return err
}
ioutil.WriteFile("routes.json", data, 0644)
```

`routes.json`

```json
[
  {
    "method": "POST",
    "path": "/users",
    "handler": "main.createUser"
  },
  {
    "method": "GET",
    "path": "/users",
    "handler": "main.findUser"
  },
  {
    "method": "PUT",
    "path": "/users",
    "handler": "main.updateUser"
  },
  {
    "method": "DELETE",
    "path": "/users",
    "handler": "main.deleteUser"
  }
]
```