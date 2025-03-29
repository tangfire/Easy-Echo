# 类似 Twitter 的 API 服务

这个示例演示如何使用 MongoDB，JWT 和 JSON 创建一个类似 Twitter 的 REST API 服务。

## 模型


`user.go`

```go
package model

import "gopkg.in/mgo.v2/bson"

type (
    User struct {
        ID        bson.ObjectId `json:"id" bson:"_id,omitempty"`
        Email     string        `json:"email" bson:"email"`
        Password  string        `json:"password,omitempty" bson:"password"`
        Token     string        `json:"token,omitempty" bson:"-"`
        Followers []string      `json:"followers,omitempty" bson:"followers,omitempty"`
    }
)
```


这段代码定义了一个与 **MongoDB** 交互的 `User` 结构体模型，通过 `bson` 标签控制数据存储字段映射，通过 `json` 标签控制 API 响应字段序列化。以下是逐字段解析：

---

### **1. 结构体字段与标签解析**
```go
type User struct {
    ID        bson.ObjectId `json:"id" bson:"_id,omitempty"`
    Email     string        `json:"email" bson:"email"`
    Password  string        `json:"password,omitempty" bson:"password"`
    Token     string        `json:"token,omitempty" bson:"-"`
    Followers []string      `json:"followers,omitempty" bson:"followers,omitempty"`
}
```

#### **(1) `ID` 字段**
- **类型**：`bson.ObjectId`  
  表示 MongoDB 的默认主键类型 `_id`，是一个 24 位的 16 进制字符串（如 `5a66a96306d2a40a8b884049`）。
- **标签**：
    - `bson:"_id,omitempty"`：将 Go 字段映射到 MongoDB 的 `_id` 字段，`omitempty` 表示如果字段为空（如新建用户时未指定 `ID`），MongoDB 会自动生成一个 `ObjectId`。
    - `json:"id"`：在 JSON 响应中，`_id` 字段以 `id` 为键名返回（如 `{"id": "5a66a963...", ...}`）。

#### **(2) `Email` 与 `Password` 字段**
- **标签**：
    - `bson:"email"` 和 `bson:"password"`：直接映射到 MongoDB 的同名字段。
    - `json:"email"` 和 `json:"password,omitempty"`：`password` 的 `omitempty` 表示当该字段为空时，不序列化到 JSON 响应中（避免敏感信息泄露）。

#### **(3) `Token` 字段**
- **标签**：
    - `bson:"-"`：短横线 `-` 表示此字段不存储到 MongoDB（如临时生成的鉴权 Token 无需持久化）。
    - `json:"token,omitempty"`：仅在 Token 非空时出现在 JSON 响应中。

#### **(4) `Followers` 字段**
- **标签**：
    - `bson:"followers,omitempty"` 和 `json:"followers,omitempty"`：`omitempty` 表示当切片为空时，MongoDB 和 JSON 响应中均不包含此字段（避免存储或返回空数组 `[]`）。

---

### **2. 关键设计意图**
#### **(1) 数据存储与 API 响应的解耦**
- **`bson` 标签**：控制 MongoDB 文档的字段映射和存储行为（如主键生成规则）。
- **`json` 标签**：控制 API 响应的字段名和数据暴露策略（如隐藏密码字段）。

#### **(2) 安全性优化**
- **敏感字段处理**：`Password` 和 `Token` 使用 `omitempty` 避免泄露，符合最小数据暴露原则。
- **主键映射**：MongoDB 的 `_id` 在 API 中重命名为 `id`，避免暴露内部实现细节。

---

### **3. 实际使用场景示例**
#### **(1) 创建用户**
```go
user := User{
    Email:    "user@example.com",
    Password: "hashed_password",
}
// MongoDB 插入时自动生成 ID
err := db.C("users").Insert(&user)
```
- **结果存储**：MongoDB 中生成 `_id` 字段，`Token` 字段被忽略。

#### **(2) 查询用户**
```go
var user User
db.C("users").Find(bson.M{"email": "user@example.com"}).One(&user)
```
- **JSON 响应**：
  ```json
  {
    "id": "5a66a96306d2a40a8b884049",
    "email": "user@example.com",
    "followers": ["user2"]
  }
  ```
    - `password` 和空 `token` 字段被隐藏。

---

### **4. 扩展建议**
- **字段校验**：可在结构体添加 `validate` 标签（如 `validate:"required,email"`）配合校验库（如 `go-playground/validator`）实现数据合法性检查。
- **加密存储**：`Password` 字段应存储哈希值而非明文，推荐使用 `bcrypt` 或 `argon2` 算法。
- **索引优化**：对高频查询字段（如 `Email`）添加 MongoDB 索引以加速查询。

此代码是 MongoDB 数据模型的典型实现，兼顾了存储效率与 API 安全性。


`post.go`

```go
package model

import "gopkg.in/mgo.v2/bson"

type (
    Post struct {
        ID      bson.ObjectId `json:"id" bson:"_id,omitempty"`
        To      string        `json:"to" bson:"to"`
        From    string        `json:"from" bson:"from"`
        Message string        `json:"message" bson:"message"`
    }
)
```


## 控制器

`handler.go`

```go
package handler

import mgo "gopkg.in/mgo.v2"

type (
    Handler struct {
        DB *mgo.Session
    }
)

const (
    // Key (Should come from somewhere else).
    Key = "secret"
)
```

`user.go`

```go
package handler

import (
    "net/http"
    "time"

    jwt "github.com/dgrijalva/jwt-go"
    "github.com/labstack/echo"
    "github.com/labstack/echo/cookbook/twitter/model"
    mgo "gopkg.in/mgo.v2"
    "gopkg.in/mgo.v2/bson"
)

func (h *Handler) Signup(c echo.Context) (err error) {
    // Bind
    u := &model.User{ID: bson.NewObjectId()}
    if err = c.Bind(u); err != nil {
        return
    }

    // Validate
    if u.Email == "" || u.Password == "" {
        return &echo.HTTPError{Code: http.StatusBadRequest, Message: "invalid email or password"}
    }

    // Save user
    db := h.DB.Clone()
    defer db.Close()
    if err = db.DB("twitter").C("users").Insert(u); err != nil {
        return
    }

    return c.JSON(http.StatusCreated, u)
}

func (h *Handler) Login(c echo.Context) (err error) {
    // Bind
    u := new(model.User)
    if err = c.Bind(u); err != nil {
        return
    }

    // Find user
    db := h.DB.Clone()
    defer db.Close()
    if err = db.DB("twitter").C("users").
        Find(bson.M{"email": u.Email, "password": u.Password}).One(u); err != nil {
        if err == mgo.ErrNotFound {
            return &echo.HTTPError{Code: http.StatusUnauthorized, Message: "invalid email or password"}
        }
        return
    }

    //-----
    // JWT
    //-----

    // Create token
    token := jwt.New(jwt.SigningMethodHS256)

    // Set claims
    claims := token.Claims.(jwt.MapClaims)
    claims["id"] = u.ID
    claims["exp"] = time.Now().Add(time.Hour * 72).Unix()

    // Generate encoded token and send it as response
    u.Token, err = token.SignedString([]byte(Key))
    if err != nil {
        return err
    }

    u.Password = "" // Don't send password
    return c.JSON(http.StatusOK, u)
}

func (h *Handler) Follow(c echo.Context) (err error) {
    userID := userIDFromToken(c)
    id := c.Param("id")

    // Add a follower to user
    db := h.DB.Clone()
    defer db.Close()
    if err = db.DB("twitter").C("users").
        UpdateId(bson.ObjectIdHex(id), bson.M{"$addToSet": bson.M{"followers": userID}}); err != nil {
        if err == mgo.ErrNotFound {
            return echo.ErrNotFound
        }
    }

    return
}

func userIDFromToken(c echo.Context) string {
    user := c.Get("user").(*jwt.Token)
    claims := user.Claims.(jwt.MapClaims)
    return claims["id"].(string)
}
```

`post.go`

```go
package handler

import (
    "net/http"
    "strconv"

    "github.com/labstack/echo"
    "github.com/labstack/echo/cookbook/twitter/model"
    mgo "gopkg.in/mgo.v2"
    "gopkg.in/mgo.v2/bson"
)

func (h *Handler) CreatePost(c echo.Context) (err error) {
    u := &model.User{
        ID: bson.ObjectIdHex(userIDFromToken(c)),
    }
    p := &model.Post{
        ID:   bson.NewObjectId(),
        From: u.ID.Hex(),
    }
    if err = c.Bind(p); err != nil {
        return
    }

    // Validation
    if p.To == "" || p.Message == "" {
        return &echo.HTTPError{Code: http.StatusBadRequest, Message: "invalid to or message fields"}
    }

    // Find user from database
    db := h.DB.Clone()
    defer db.Close()
    if err = db.DB("twitter").C("users").FindId(u.ID).One(u); err != nil {
        if err == mgo.ErrNotFound {
            return echo.ErrNotFound
        }
        return
    }

    // Save post in database
    if err = db.DB("twitter").C("posts").Insert(p); err != nil {
        return
    }
    return c.JSON(http.StatusCreated, p)
}

func (h *Handler) FetchPost(c echo.Context) (err error) {
    userID := userIDFromToken(c)
    page, _ := strconv.Atoi(c.QueryParam("page"))
    limit, _ := strconv.Atoi(c.QueryParam("limit"))

    // Defaults
    if page == 0 {
        page = 1
    }
    if limit == 0 {
        limit = 100
    }

    // Retrieve posts from database
    posts := []*model.Post{}
    db := h.DB.Clone()
    if err = db.DB("twitter").C("posts").
        Find(bson.M{"to": userID}).
        Skip((page - 1) * limit).
        Limit(limit).
        All(&posts); err != nil {
        return
    }
    defer db.Close()

    return c.JSON(http.StatusOK, posts)
}
```

## API

## 注册

用户注册

- 用请求里取出用户信息验证合法性。
- 不合法的邮箱和密码，返回 `400 - Bad Request`。
- 合法的邮箱和密码，保存数据到数据库并返回 `201 - Created`。







## 登录

用户登录

- 从正文中检索用户凭证并针对数据库进行验证。
- 对于无效的凭据，请发送 `401 - Unauthorized` 响应。
- 对于有效凭证，请发送 `200 - OK` 响应：
  - 为用户生成 JWT 并将其作为响应发送。
  - 每个后续请求都必须在 `Authorization` 标头中包含 JWT。

方法：`POST`

路径：`/login`

### 请求

```go
curl \
  -X POST \
  http://localhost:1323/login \
  -H "Content-Type: application/json" \
  -d '{"email":"jon@labstack.com","password":"shhh!"}'

```


### 响应

`200 - 正常`

```go
{
  "id": "58465b4ea6fe886d3215c6df",
  "email": "jon@labstack.com",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE0ODEyNjUxMjgsImlkIjoiNTg0NjViNGVhNmZlODg2ZDMyMTVjNmRmIn0.1IsGGxko1qMCsKkJDQ1NfmrZ945XVC9uZpcvDnKwpL0"
}
```

客户端应该存储 Token，对于浏览器，您可以使用本地存储。

## 跟随



