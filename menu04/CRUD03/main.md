# CRUD示例

## 服务端

`server.go`

```go
package main

import (
    "net/http"
    "strconv"

    "github.com/labstack/echo"
    "github.com/labstack/echo/middleware"
)

type (
    user struct {
        ID   int    `json:"id"`
        Name string `json:"name"`
    }
)

var (
    users = map[int]*user{}
    seq   = 1
)

//----------
// Handlers
//----------

func createUser(c echo.Context) error {
    u := &user{
        ID: seq,
    }
    if err := c.Bind(u); err != nil {
        return err
    }
    users[u.ID] = u
    seq++
    return c.JSON(http.StatusCreated, u)
}

func getUser(c echo.Context) error {
    id, _ := strconv.Atoi(c.Param("id"))
    return c.JSON(http.StatusOK, users[id])
}

func updateUser(c echo.Context) error {
    u := new(user)
    if err := c.Bind(u); err != nil {
        return err
    }
    id, _ := strconv.Atoi(c.Param("id"))
    users[id].Name = u.Name
    return c.JSON(http.StatusOK, users[id])
}

func deleteUser(c echo.Context) error {
    id, _ := strconv.Atoi(c.Param("id"))
    delete(users, id)
    return c.NoContent(http.StatusNoContent)
}

func main() {
    e := echo.New()

    // Middleware
    e.Use(middleware.Logger())
    e.Use(middleware.Recover())

    // Routes
    e.POST("/users", createUser)
    e.GET("/users/:id", getUser)
    e.PUT("/users/:id", updateUser)
    e.DELETE("/users/:id", deleteUser)

    // Start server
    e.Logger.Fatal(e.Start(":1323"))
}
```


这段代码使用 **Go 的 Echo 框架**实现了一个完整的 RESTful API，支持用户数据的增删改查（CRUD）操作。以下是分模块解析：

---

### **1. 核心数据结构与全局变量**
```go
type user struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

var (
    users = map[int]*user{} // 内存存储用户数据
    seq   = 1               // 自增 ID 生成器
)
```
- **`user` 结构体**：定义用户数据模型，包含 `ID` 和 `Name` 字段，并通过 JSON 标签支持序列化。
- **内存存储**：使用全局变量 `users`（Map）和 `seq` 实现简易内存数据库，适用于轻量级场景（生产环境建议使用持久化存储如数据库）。

---

### **2. 初始化 Echo 实例与中间件**
```go
func main() {
    e := echo.New()
    e.Use(middleware.Logger())   // 记录请求日志
    e.Use(middleware.Recover())  // 捕获 panic 防止服务崩溃
    // 注册路由
    e.Logger.Fatal(e.Start(":1323"))
}
```
- **`Logger` 中间件**：记录请求方法、路径、状态码和耗时，便于调试。
- **`Recover` 中间件**：拦截未处理的异常（如空指针），返回 HTTP 500 错误，保障服务高可用。
- **Echo 启动**：监听 1323 端口，若端口冲突或启动失败，通过 `Logger.Fatal` 终止程序。

---

### **3. 路由与 HTTP 方法处理**
#### **(1) 创建用户（POST /users）**
```go
func createUser(c echo.Context) error {
    u := &user{ID: seq}
    if err := c.Bind(u); err != nil { // 解析请求体到 user 结构体
        return err
    }
    users[u.ID] = u
    seq++
    return c.JSON(http.StatusCreated, u) // 返回 JSON 响应
}
```
- **数据绑定**：`c.Bind()` 自动解析请求体（如 JSON）并填充 `user` 对象。
- **响应状态码**：`201 Created` 表示资源创建成功，符合 REST 规范。

#### **(2) 获取用户（GET /users/:id）**
```go
func getUser(c echo.Context) error {
    id, _ := strconv.Atoi(c.Param("id")) // 从路径参数获取 ID
    return c.JSON(http.StatusOK, users[id])
}
```
- **动态路由**：`:id` 表示路径参数，通过 `c.Param("id")` 提取。
- **简化的错误处理**：代码中未处理 `strconv.Atoi` 的潜在错误（如非数字 ID），实际需补充校验逻辑。

#### **(3) 更新用户（PUT /users/:id）**
```go
func updateUser(c echo.Context) error {
    u := new(user)
    if err := c.Bind(u); err != nil {
        return err
    }
    id, _ := strconv.Atoi(c.Param("id"))
    users[id].Name = u.Name // 直接修改内存中的用户数据
    return c.JSON(http.StatusOK, users[id])
}
```
- **数据更新**：通过路径参数定位用户，覆盖 `Name` 字段。
- **幂等性**：PUT 方法要求请求多次效果相同，符合 REST 设计原则。

#### **(4) 删除用户（DELETE /users/:id）**
```go
func deleteUser(c echo.Context) error {
    id, _ := strconv.Atoi(c.Param("id"))
    delete(users, id)
    return c.NoContent(http.StatusNoContent) // 204 表示无返回内容
}
```
- **删除操作**：使用 Go 内置的 `delete()` 函数从 Map 中移除用户。
- **状态码 204**：表示操作成功且无需返回数据。

---

### **4. 性能与扩展性**
- **路由效率**：Echo 的路由基于 **Trie 树**（前缀树），支持快速路径匹配，尤其适合动态路由场景（如 `/users/:id`）。
- **中间件链**：注册的中间件按顺序执行，可在请求处理前后插入逻辑（如日志、鉴权）。
- **并发安全**：当前代码未对全局变量 `users` 和 `seq` 加锁，多线程环境下可能引发竞态条件，需通过互斥锁（`sync.Mutex`）或改用并发安全的数据结构优化。

---

### **代码整体作用**
1. **提供 RESTful API**：支持通过标准 HTTP 方法（POST/GET/PUT/DELETE）管理用户数据。
2. **数据持久化**：内存存储简化了示例，便于快速验证功能。
3. **安全性**：通过中间件实现基础防护（如异常恢复），但需补充身份验证（如 JWT）保障敏感操作。

---

### **扩展建议**
- **持久化存储**：替换 `users` 为数据库（如 MySQL、Redis）以支持数据持久化。
- **错误处理**：完善参数校验（如 ID 是否为数字）、用户不存在时的 `404 Not Found` 响应。
- **API 文档**：集成 Swagger 自动生成接口文档，提升易用性。

## 客户端

`curl`

### 创建User

```go
curl -X POST \
  -H 'Content-Type: application/json' \
  -d '{"name":"Joe Smith"}' \
  localhost:1323/users
```

Response

```go
{
  "id": 1,
  "name": "Joe Smith"
}
```


### 获取User


```go
curl localhost:1323/users/1
```

Response

```go
{
  "id": 1,
  "name": "Joe Smith"
}
```

### 更新User

```go
curl -X PUT \
  -H 'Content-Type: application/json' \
  -d '{"name":"Joe"}' \
  localhost:1323/users/1
```

Response

```go
{
  "id": 1,
  "name": "Joe"
}
```

### 删除User

```go
curl -X DELETE localhost:1323/users/1
```

Response 

`NoContent - 204`


