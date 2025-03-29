# Gzip

## Gzip中间件

Gzip中间件使用gzip方案对HTTP响应进行压缩

用法

`e.Use(middleware.Gzip())`

## 自定义配置

用法

```go
e := echo.New()
e.Use(middleware.GzipWithConfig(middleware.GzipConfig{
  Level: 5,
}))
```

## 配置

```go
GzipConfig struct {
  // Skipper defines a function to skip middleware.
  Skipper Skipper

  // Gzip compression level.
  // Optional. Default value -1.
  Level int `json:"level"`
}
```

默认配置

```go
DefaultGzipConfig = GzipConfig{
  Skipper: defaultSkipper,
  Level:   -1,
}
```

