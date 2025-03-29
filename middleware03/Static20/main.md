# Static (静态) 中间件

Static 中间件可已被用于服务从根目录中提供的静态文件。

用法

```go
e := echo.New()
e.Use(middleware.Static("/static"))
```

上例为 `static` 目录下的静态文件提供服务。例如， 一个 `/js/main.js` 的请求将捕获并服务 `static/js/main.js` 文件。


# 配置

```go
StaticConfig struct {
  // Skipper defines a function to skip middleware.
  Skipper Skipper

  // Root directory from where the static content is served.
  // Required.
  Root string `json:"root"`

  // Index file for serving a directory.
  // Optional. Default value "index.html".
  Index string `json:"index"`

  // Enable HTML5 mode by forwarding all not-found requests to root so that
  // SPA (single-page application) can handle the routing.
  // Optional. Default value false.
  HTML5 bool `json:"html5"`

  // Enable directory browsing.
  // Optional. Default value false.
  Browse bool `json:"browse"`
}
```

默认配置

```go
DefaultStaticConfig = StaticConfig{
  Skipper: DefaultSkipper,
  Index:   "index.html",
}
```