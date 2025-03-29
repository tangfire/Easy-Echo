# Logger (日志) 中间件

Logger 中间件记录有关每个 HTTP 请求的信息。

用法

```go
e.Use(middleware.Logger())
```

输出样例

```go
{"time":"2017-01-12T08:58:07.372015644-08:00","remote_ip":"::1","host":"localhost:1323","method":"GET","uri":"/","status":200, "latency":14743,"latency_human":"14.743µs","bytes_in":0,"bytes_out":2}
```

# 自定义配置

用法

```go
e.Use(middleware.LoggerWithConfig(middleware.LoggerConfig{
  Format: "method=${method}, uri=${uri}, status=${status}\n",
}))
```

上面的示例使用 `Format` 来记录请求方法和请求 URI 。

输出样例

```go
method=GET, uri=/, status=200

```

# 配置

```go
LoggerConfig struct {
  // Skipper 定义了一个跳过中间件的函数.
  Skipper Skipper

  // 日志的格式可以使用下面的标签定义。:
  //
  // - time_unix
  // - time_unix_nano
  // - time_rfc3339
  // - time_rfc3339_nano
  // - id (Request ID - Not implemented)
  // - remote_ip
  // - uri
  // - host
  // - method
  // - path
  // - referer
  // - user_agent
  // - status
  // - latency (In microseconds)
  // - latency_human (Human readable)
  // - bytes_in (Bytes received)
  // - bytes_out (Bytes sent)
  // - header:<name>
  // - query:<name>
  // - form:<name>
  //
  // 例如 "${remote_ip} ${status}"
  //
  // 可选。默认值是 DefaultLoggerConfig.Format.
  Format string `json:"format"`

  // Output 是记录日志的位置。
  // 可选。默认值是 os.Stdout.
  Output io.Writer
}
```

默认配置

```go
DefaultLoggerConfig = LoggerConfig{
  Skipper: defaultSkipper,
  Format: `{"time":"${time_rfc3339_nano}","remote_ip":"${remote_ip}","host":"${host}",` +
    `"method":"${method}","uri":"${uri}","status":${status}, "latency":${latency},` +
    `"latency_human":"${latency_human}","bytes_in":${bytes_in},` +
    `"bytes_out":${bytes_out}}` + "\n",
  Output: os.Stdout
}
```




