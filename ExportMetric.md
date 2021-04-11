Prometheus客户端不能直接将指标发送给Prometheus服务器。只能基于HTTP协议暴露，将promhttp包提供的Handler传递给你的HTTP服务器，每个Handler读取单个注册表，从中收集指标信息，生成HTTP响应：
```go
package main

import (
   "net/http"
   "github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
   http.Handle("/metrics", promhttp.Handler())
   http.ListenAndServe(":2112", nil)
}
```
访问`http://localhost:2112/metrics`, 获取指标如下

```
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 8
。。。。。省略其他指标
# HELP go_threads Number of OS threads created.
# TYPE go_threads gauge
go_threads 7
# HELP promhttp_metric_handler_requests_in_flight Current number of scrapes being served.
# TYPE promhttp_metric_handler_requests_in_flight gauge
promhttp_metric_handler_requests_in_flight 1
```
可以看到我们并没有注册自己的指标，而是获取的一些默认指标
```go
// 使用默认注册表创建Handler
func Handler() http.Handler {
    return InstrumentMetricHandler(
        prometheus.DefaultRegisterer, HandlerFor(prometheus.DefaultGatherer, HandlerOpts{}),
    )
}
```
如果需要使用非默认注册表，可以使用如下方法
```go
package main

import (
   "github.com/prometheus/client_golang/prometheus"
   "github.com/prometheus/client_golang/prometheus/promhttp"
   "net/http"
)

var (
   cpuTemp = prometheus.NewGauge(prometheus.GaugeOpts{
      Name: "cpu_temperature_celsius",
      Help: "Current temperature of the CPU.",
   })
)

func main() {
   reg := prometheus.NewPedanticRegistry()
   reg.MustRegister(cpuTemp)
   gatherers := prometheus.Gatherers{
   // prometheus.DefaultGatherer, // 添加默认指标
      reg,
   }

   h := promhttp.HandlerFor(gatherers,
      promhttp.HandlerOpts{})
   http.HandleFunc("/metrics", func(w http.ResponseWriter, r *http.Request) {
      h.ServeHTTP(w, r)
   })

   http.ListenAndServe(":2112", nil)
}
```
访问`http://localhost:2112/metrics`, 获取指标如下
```
# HELP cpu_temperature_celsius Current temperature of the CPU.
# TYPE cpu_temperature_celsius gauge
cpu_temperature_celsius 0
```
