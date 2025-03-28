# 响应

## 发送string数据

`Context#String(code int, s string)` 用于发送一个带有状态码的纯文本响应。

示例

```go
func(c echo.Context) error {
  return c.String(http.StatusOK, "Hello, World!")
}
```

## 发送 HTML 响应 (参考模板)

`Context#HTML(code int, html string)` 用于发送一个带有状态码的简单 HTML 响应。

示例

```go
func(c echo.Context) error {
  return c.HTML(http.StatusOK, "<strong>Hello, World!</strong>")
}
```


## 发送 HTML Blob

`Context#HTMLBlob(code int, b []byte)`用于发送一个带状态码的 HTML blob（二进制长对象）响应。可以发现，使用输出 `[]byte` 的模版引擎很方便。

## 模版引擎渲染

略

## 发送JSON数据

`Context#JSON(code int, i interface{})` 用于发送一个带状态码的 JSON 对象，它会将 Golang 的对象转换成 JSON 字符串。

示例

```go
// User
type User struct {
  Name  string `json:"name" xml:"name"`
  Email string `json:"email" xml:"email"`
}

// Handler
func(c echo.Context) error {
  u := &User{
    Name:  "Jon",
    Email: "jon@labstack.com",
  }
  return c.JSON(http.StatusOK, u)
}
```

## JSON流

`Context#JSON()` 内部使用 `json.Marshal` 来转换 JSON 数据，但该方法面对大量的 JSON 数据会显得效率不足，对于这种情况可以直接使用 JSON 流。


示例

```go
func(c echo.Context) error {
  u := &User{
    Name:  "Jon",
    Email: "jon@labstack.com",
  }
  c.Response().Header().Set(echo.HeaderContentType, echo.MIMEApplicationJSONCharsetUTF8)
  c.Response().WriteHeader(http.StatusOK)
  return json.NewEncoder(c.Response()).Encode(u)
}
```

以下是对这段代码的逐层解析：

---

### 一、代码功能概述
这是一个基于Echo框架的Go语言HTTP处理函数，主要功能是手动构建一个JSON格式的HTTP响应，返回用户数据`User`结构体的JSON序列化结果，并设置相关响应头信息。

---

### 二、代码逐行解析
```go
func(c echo.Context) error {
    // 1. 创建User结构体实例并初始化字段
    u := &User{
        Name:  "Jon",
        Email: "jon@labstack.com",
    }
    
    // 2. 设置响应头Content-Type为JSON格式（含UTF-8字符集）
    c.Response().Header().Set(echo.HeaderContentType, echo.MIMEApplicationJSONCharsetUTF8)
    
    // 3. 设置HTTP状态码为200
    c.Response().WriteHeader(http.StatusOK)
    
    // 4. 将结构体序列化为JSON并写入响应体
    return json.NewEncoder(c.Response()).Encode(u)
}
```

---

### 三、关键技术点详解
#### 1. 手动设置响应头
- `echo.HeaderContentType`：Echo预定义的常量，值为`"Content-Type"`
- `echo.MIMEApplicationJSONCharsetUTF8`：预定义常量，值为`"application/json; charset=UTF-8"`
- **作用**：明确告知客户端返回的是JSON格式数据，避免浏览器错误解析

#### 2. 状态码设置
- `WriteHeader(http.StatusOK)`显式设置200状态码，与框架提供的`c.JSON(http.StatusOK, u)`等价，但这里采用底层操作实现

#### 3. JSON序列化方式
- `json.NewEncoder().Encode()`：流式序列化方法，直接将结构体编码到响应流
- 与`c.JSON()`的区别：
    - **标准方式**：`return c.JSON(http.StatusOK, u)`（自动设置Content-Type和状态码）
    - **手动方式**：允许更细粒度的控制（如自定义日期格式、空值处理等）

---

### 四、与Echo常规写法的对比
常规简化写法：
```go
func(c echo.Context) error {
    u := &User{Name: "Jon", Email: "jon@labstack.com"}
    return c.JSON(http.StatusOK, u)
}
```
两种实现的核心差异：  
| 特性               | 手动方式                     | 自动方式(c.JSON)         |
|--------------------|----------------------------|-------------------------|
| 响应头设置          | 需手动指定                 | 自动完成                |
| 状态码设置          | 需显式调用WriteHeader      | 通过参数自动设置        |
| 适用场景            | 需要定制序列化逻辑时       | 快速开发标准API时       |

---

### 五、相关技术延伸
1. **结构体标签的作用**  
   如果`User`结构体包含验证标签（如`validate:"required"`），需配合验证中间件使用，但本示例未包含验证逻辑

2. **性能考量**
    - 流式编码（`NewEncoder`）比`json.Marshal`更节省内存，适合大对象响应
    - Echo的`c.JSON()`内部也使用`json.NewEncoder`实现

3. **错误处理**  
   `Encode()`方法返回的error会被Echo框架自动捕获，最终生成500错误响应

---

### 六、完整请求处理流程
1. 客户端发送HTTP请求
2. Echo路由匹配到该处理函数
3. 创建User对象
4. 设置响应头Content-Type
5. 写入200状态码
6. 流式编码User对象到响应体
7. 框架自动处理错误并发送响应

---

通过这种手动控制方式，开发者可以灵活处理特殊序列化需求（如分块传输、自定义时间格式等），但常规开发更推荐使用`c.JSON()`简化代码。


## JSON 美化 (JSON Pretty)

`Context#JSONPretty(code int, i interface{}, indent string)` 可以发送带有缩进（可以使用空格和 tab）的更为好看的 JSON 数据。

示例

```go
func(c echo.Context) error {
  u := &User{
    Name:  "Jon",
    Email: "joe@labstack.com",
  }
  return c.JSONPretty(http.StatusOK, u, "  ")
}
```

```go
{
  "email": "joe@labstack.com",
  "name": "Jon"
}
```

通过在请求URL查询字符串中附加 `pretty` ，你也可以使用 `Context#JSON()` 来输出带有缩进的 JSON 数据。

示例

```go
curl http://localhost:1323/users/1?pretty
```

## JSON Blob

`Context#JSONBlob(code int, b []byte)` 可用来从外部源（例如数据库）直接发送预编码的 JSON 对象。****

示例

```go
func(c echo.Context) error {
  encodedJSON := []byte{} // Encoded JSON from external source
  return c.JSONBlob(http.StatusOK, encodedJSON)
}
```

# 发送JSONP数据

`Context#JSONP(code int, callback string, i interface{})` 可以将 Golang 的数据类型转换成 JSON 类型，并通过回调以带有状态码的 JSONNP 结构发送。

# 发送XML数据

`Context#XML(code int, i interface{})` 可以将 Golang 对象转换成 XML 类型，并带上状态码发送响应。

示例

```go
func(c echo.Context) error {
  u := &User{
    Name:  "Jon",
    Email: "jon@labstack.com",
  }
  return c.XML(http.StatusOK, u)
}
```

## XML流

`Context#XML` 内部使用 `xml.Marshal` 来转换 XML 数据，但该方法面对大量的 XML 数据会显得效率不足，对于这种情况可以直接使用 XML 流。

示例

```go
func(c echo.Context) error {
  u := &User{
    Name:  "Jon",
    Email: "jon@labstack.com",
  }
  c.Response().Header().Set(echo.HeaderContentType, echo.MIMEApplicationXMLCharsetUTF8)
  c.Response().WriteHeader(http.StatusOK)
  return xml.NewEncoder(c.Response()).Encode(u)
}
```

## XML 美化 (XML Pretty)

`Context#XMLPretty(code int, i interface{}, indent string)` 可以发送带有缩进（可以使用空格和 tab）的更为好看的 XML 数据。

示例

```go
func(c echo.Context) error {
  u := &User{
    Name:  "Jon",
    Email: "joe@labstack.com",
  }
  return c.XMLPretty(http.StatusOK, u, "  ")
}
```

```html
<?xml version="1.0" encoding="UTF-8"?>
<User>
  <Name>Jon</Name>
  <Email>joe@labstack.com</Email>
</User>
```


通过在请求URL查询字符串中附加 `pretty` ，你也可以使用 `Context#XML()` 来输出带有缩进的 XML 数据。

示例

```go
curl http://localhost:1323/users/1?pretty
```


## XML Blob

`Context#XMLBlob(code int, b []byte)` 可用来从外部源（例如数据库）直接发送预编码的 XML 对象。

示例

```go
func(c echo.Context) error {
  encodedXML := []byte{} // Encoded XML from external source
  return c.XMLBlob(http.StatusOK, encodedXML)
}
```

## 发送文件

`Context#File(file string)` 可用来发送内容为文件的响应，并且它能自动设置正确的内容类型、优雅地处理缓存。

示例

```go
func(c echo.Context) error {
  return c.File("<文件路径>")
}
```

## 发送附件

`Context#Attachment(file, name string)` 和发送文件 `File()` 的方法类似，只是它的方法名称不同。

示例

```go
func(c echo.Context) error {
  return c.Attachment("<PATH_TO_YOUR_FILE>")
}
```

## 发送内嵌 (Inline)

`Context#Inline(file, name string)` 和发送文件 `File()` 的方法类似，只是它的方法名称不同。

示例

```go
func(c echo.Context) error {
  return c.Inline("<PATH_TO_YOUR_FILE>")
}
```

## 发送二进制长文件 (Blob)

`Context#Blob(code int, contentType string, b []byte)` 可用于发送带有内容类型 (content type) 和状态代码的任意类型数据。

示例

```go
func(c echo.Context) (err error) {
  data := []byte(`0306703,0035866,NO_ACTION,06/19/2006
      0086003,"0005866",UPDATED,06/19/2006`)
    return c.Blob(http.StatusOK, "text/csv", data)
}
```

## 发送流(Stream)

`Context#Stream(code int, contentType string, r io.Reader)` 可用于发送带有内容类型 (content type) 、状态代码、`io.Reader` 的任意类型数据流。

示例

```go
func(c echo.Context) error {
  f, err := os.Open("<PATH_TO_IMAGE>")
  if err != nil {
    return err
  }
  return c.Stream(http.StatusOK, "image/png", f)
}
```

## 发送空内容 (No Content)

`Context#NoContent(code int)` 可用于发送带有状态码的空内容。

示例


```go
func(c echo.Context) error {
  return c.NoContent(http.StatusOK)
}
```

## 重定向

`Context#Redirect(code int, url string)` 可用于重定向至一个带有状态码的 URL。

示例

```go
func(c echo.Context) error {
  return c.Redirect(http.StatusMovedPermanently, "<URL>")
}
```

## Hooks

### 响应之前

`Context#Response#Before(func())` 可以用来注册在写入响应之前调用的函数。

### 响应之后

`Context#Response#After(func())` 可以用来注册在写入响应之后调用的函数。但是如果 “Content-Length” 是未知状态，则不会有任何方法会被执行。

示例

```go
func(c echo.Context) error {
  c.Response().Before(func() {
    println("before response")
  })
  c.Response().After(func() {
    println("after response")
  })
  return c.NoContent(http.StatusNoContent)
}
```

可以在响应之前与之后注册多个方法

