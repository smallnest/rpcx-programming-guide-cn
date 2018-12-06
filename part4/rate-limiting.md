# 限流

**实例:** [rate-limiting](https://github.com/smallnest/rpcx/blob/master/serverplugin/rate_limiting.go)

限流是一种保护错误，避免服务被突发的或者大量的请求所拖垮。

这个插件使用 [juju/ratelimit](https://github.com/juju/ratelimit)来限流。

使用 `func NewRateLimitingPlugin(fillInterval time.Duration, capacity int64) *RateLimitingPlugin` t来创建这个插件。
