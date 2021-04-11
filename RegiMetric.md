上节我们定义了两个指标
```go
 cpuTemp = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "cpu_temperature_celsius",
        Help: "Current temperature of the CPU.",
    })
    hdFailures = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "hd_errors_total",
            Help: "Number of hard-disk errors.",
        },
        []string{"device"},
    )
```
指标必须被注册才能被收集，注册方式如下
```go
func init() {
    // Metrics have to be registered to be exposed:
    prometheus.MustRegister(cpuTemp)
    prometheus.MustRegister(hdFailures)
}
```
使用prometheus.MustRegister是将数据直接注册到Default Registry，就像上面的运行的例子一样，这个Default Registry不需要额外的任何代码就可以将指标传递出去。注册后既可以在程序层面上去使用该指标了，这里我们使用之前定义的指标提供的API（Set和With().Inc）去改变指标的数据内容
```go
func main() {
    cpuTemp.Set(65.3)
    hdFailures.With(prometheus.Labels{"device":"/dev/sda"}).Inc()


    // The Handler function provides a default handler to expose metrics
    // via an HTTP server. "/metrics" is the usual endpoint for that.
    http.Handle("/metrics", promhttp.Handler())
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```
**注意**：MustRegister 是注册collector最通用的方式。如果需要捕获注册时产生的错误，可以使用Register 函数，该函数会返回错误。
```go
// MustRegister implements Registerer.
func (r *Registry) MustRegister(cs ...Collector) {
   for _, c := range cs {
      if err := r.Register(c); err != nil {
         panic(err)
      }
   }
}
```
以上提到的registry都被称为默认registry，可以在全局变量DefaultRegisterer中找到。
DefaultRegisterer注册了Go runtime metrics （通过NewGoCollector）和用于process metrics 的collector（通过NewProcessCollector）。
```go
var (
   defaultRegistry              = NewRegistry()
   DefaultRegisterer Registerer = defaultRegistry
   DefaultGatherer   Gatherer   = defaultRegistry
)

func init() {
   MustRegister(NewProcessCollector(ProcessCollectorOpts{}))
   MustRegister(NewGoCollector())
}
```
使用NewRegistry可以创建custom registry，或者可以自己实现Registerer 或Gatherer接口。

你可以调用以下函数来创建注册表
```go
// 不预先注册任何收集器的注册表
func NewRegistry() *Registry {
    return &Registry{
        collectorsByID:  map[uint64]Collector{},
        descIDs:         map[uint64]struct{}{},
        dimHashesByName: map[string]uint64{},
    }
}
 
// 创建一个严格的注册表。该注册表在收集期间，检查
// 1、每个指标是否和它的Desc一致
// 2、指标的Desc是否已经注册到注册表
// Unchecked的收集器不被检查
func NewPedanticRegistry() *Registry {
    r := NewRegistry()
    r.pedanticChecksEnabled = true
    return r
}
registry = prometheus.NewPedanticRegistry()
```
注册收集器
```go
// 进行注册
if err := registry.Register(gauge); err != nil {

// 如果已经注册
    if are, ok := err.(prometheus.AlreadyRegisteredError); ok {

// 返回先前注册的收集器
        return are.ExistingCollector, nil

    }
    return nil, err

} 
```
Registry注册表，此结构实现了Registerer、Gatherer接口。
Registerer接口为注册表提供注册/反注册功能：

```go
type Registerer interface {

    // 注册一个需要包含在指标集中的收集器。如果收集器提供的描述符非法、
    // 或者不满足metric.Desc的一致性/唯一性需求，则返回错误
    //
    // 如果相等的收集器已经注册过，返回AlreadyRegisteredError，其中包含先前注册的收集器的实例
    //
    // 其Describe方法不产生任何Desc的收集器，视为Unchecked，对这种收集器的注册总是成功
    // 重现注册它时也不会有检查。因此，调用者必须负责确保不会重复注册
    Register(Collector) error

    // 注册多个收集器，并且在遇到第一个失败时就Panic
    MustRegister(...Collector)
    // 反注册
    Unregister(Collector) bool

}
```
Gatherer为注册表提供汇集（gathering）功能 —— 将已经收集的指标汇集到若干指标族（MetricFamily）中
```go
type Gatherer interface {

    // 该方法调用所有已经注册的收集器的Collect方法，然后将获得的指标存放到一个字典序排列
    // 的MetricFamily的切片中。该方法保证返回的切片是有效的、自我一致的，可以用于对外
    // 暴露（给Prometheus服务器）该方法容忍相同指标族中具有不同标签集的指标
    //
    // 即时发生错误，该方法也会尝试尽可能收集更多的指标。因此，当该方法返回非空error时
    // 同时返回的dto.MetricFamily切片可能是nil（意味着致命错误）或者包含一定数量的
    // MetricFamily —— 切片可能是不完整的
    Gather() ([]*dto.MetricFamily, error)

}
```
[收集指标](GatherMetric.md)
