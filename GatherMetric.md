任何Prometheus用来收集指标的对象，都需要实现Collector接口
```go
type Collector interface {
    // 将此收集器收集的指标的所有可能的描述符发送到参数提供的通道。并且在最后一个描述符
    // 发送成功后返回。发送的描述符必须满足Desc文档声明的一致性、唯一性要求
    //
    // 同一收集器发送重复的描述符是允许的，重复自动忽视
    // 但是两个收集器不得发送重复的描述符
    //
    // 如果不发送任何描述符，则收集器标记为unchecked状态，也就是说在注册时，不会进行任何检查
    // 收集器以后可能产生任何匹配它的Collect方法签名的指标
    //
    // 该方法在收集器的生命周期里，幂等的发送相同的描述符
    //
    // 该方法可能被并发的调用，实现时需要注意线程安全问题
    //
    // 如果在执行该方法的过程中收集器遇到错误，务必发送一个无效的描述符（NewInvalidDesc）来提示注册表
    Describe(chan<- *Desc)
    // 在收集指标时，该方法被Prometheus注册表（Registry）调用。方法的实现必须将所有它收集到的指标
    // 经由参数提供的通道发送，并且在最后一个指标发送后返回
    //
    // 每个发送的指标的描述符，必须是Describe方法提供的之一（除非收集器是Unchecked）
    // 发送的共享相同描述符的指标，其标签集必须有所不同
    //
    //
    // 该方法可能被并发的调用，实现时需要注意线程安全问题
    //
    // 阻塞会导致影响所有已注册的指标的渲染性能，理想情况下，实现应该支持并发读
    Collect(chan<- Metric)
}
```
Desc该结构是任何Prometheus指标都需要使用的描述符，它本质上是指标（Metrics）的不可变元数据
```go
struct {
    // 全限定名称，由命名空间 - 子系统 - 名称组成
    fqName string

    // 指标的帮助信息
    help string

    // 常量标签键值
    constLabelPairs []*dto.LabelPair

    // 可变标签的名字
    variableLabels []string

    // 基于ConstLabels和fqName生成的哈希，所有注册的Desc都必须具有独特的值
        // 以作为Desc的唯一标识
    id uint64

    // 维度哈希，所有常量/可变标签的哈希，所有具有相同fqName的Desc必须具有相同的dimHash
    // 这意味着每个fqName对应的标签集是固定的
    dimHash uint64

    // 构造时出现的错误，注册时报告
    err error
}
```
Metric所有指标的通用接口，表示需要导出到Prometheus的单个采样值+关联的元数据集 。此接口的实现包括Gauge、Counter、Histogram、Summary、Untyped
```go
type Metric interface {

    // 幂等的返回该指标的、不可变的描述符
    // 不能描述子集的指标，必须返回一个无效的描述符。无效描述符通过NewInvalidDesc创建
    Desc() *Desc

    // 将指标对象编码为ProtoBuffer数据传输对象
    // 指标实现必须考虑并发安全性，因为对指标的读可能随时发生，任何阻塞操作都会影响
    // 所有已经注册的指标的整体渲染性能
    // 理想的实现应该支持并发读
    //
    // 除了产生dto.Metric，实现还负责确保Metric的ProtoBuf合法性验证
    // 建议使用字典序排序标签，LabelPairSorter可能对指标实现者有帮助
    Write(*dto.Metric) error

}
```
上节定义了指标，并注册了指标，但是指标并不会更新，如果想要每次请求的时候，指标进行更新，需要实现Collector的 Describe(chan<- *Desc)和Collect(chan<- Metric)方法

下面这个例子中实现了一个自定义的，满足采集器(Collector)接口的结构体，并手动注册该结构体后，使其每次查询的时候自动执行采集任务。

 - 了解了接口的实现后，我们就可以写自己的实现了，先定义结构体
```go
type Node struct {
   CpuTemp    prometheus.Gauge
   HdFailures *prometheus.CounterVec
}

func NewNode() *Node {
   return &Node{
      CpuTemp:    cpuTemp,
      HdFailures: hdFailures,
   }
}
```
 - 然后实现 Describe(chan<- *Desc)和Collect(chan<- Metric)方法
```go
// Describe implements prometheus.Collector.
func (e *Node) Describe(ch chan<- *prometheus.Desc) {
   ch <- e.CpuTemp.Desc()
   e.HdFailures.Describe(ch)
}


// Collect implements prometheus.Collector.
func (e *Node) Collect(ch chan<- prometheus.Metric) {
   cpuTemp.Add(65.3)
   ch <- e.CpuTemp


   e.HdFailures.With(prometheus.Labels{"device": "/dev/sda"}).Inc()
   e.HdFailures.Collect(ch)
}
```
 - 注册指标收集器
```go
func main() {
   node := NewNode()
   // 因为node实现了Collector接口，所以可以把node传入func MustRegister(cs ...Collector)
   prometheus.MustRegister(node)

   http.Handle("/metrics", promhttp.Handler())
   http.ListenAndServe(":2112", nil)
}
```
 - 完整的程序如下
```go
package main


import (
   "net/http"

   "github.com/prometheus/client_golang/prometheus"
   "github.com/prometheus/client_golang/prometheus/promhttp"
)


var (
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
)


func main() {
   node := NewNode()
   // 因为node实现了Collector接口，所以可以把node传入func MustRegister(cs ...Collector)
   prometheus.MustRegister(node)


   http.Handle("/metrics", promhttp.Handler())
   http.ListenAndServe(":2112", nil)
}


type Node struct {
   CpuTemp    prometheus.Gauge
   HdFailures *prometheus.CounterVec
}


func NewNode() *Node {
   return &Node{
      CpuTemp:    cpuTemp,
      HdFailures: hdFailures,
   }
}


// Describe implements prometheus.Collector.
func (e *Node) Describe(ch chan<- *prometheus.Desc) {
   ch <- e.CpuTemp.Desc()
   e.HdFailures.Describe(ch)
}


// Collect implements prometheus.Collector.
func (e *Node) Collect(ch chan<- prometheus.Metric) {
   cpuTemp.Add(65.3)
   ch <- e.CpuTemp


   e.HdFailures.With(prometheus.Labels{"device": "/dev/sda"}).Inc()
   e.HdFailures.Collect(ch)
}
```
 - 再来看下面的一直方法，体会两种的不同和适用场景
```go
package main


import (
   "net/http"


   "github.com/prometheus/client_golang/prometheus"
   "github.com/prometheus/client_golang/prometheus/promhttp"
)


var (
   cpuTempDesc = prometheus.NewDesc(
      "cpu_temperature_celsius",
      "Current temperature of the CPU.",
      []string{},
      nil,
   )
   hdFailuresDesc = prometheus.NewDesc(
      "hd_errors_total",
      "Number of hard-disk errors.",
      []string{"device"},
      nil,
   )
)


func main() {
   node := NewNode()
   // 因为node实现了Collector接口，所以可以把node传入func MustRegister(cs ...Collector)
   prometheus.MustRegister(node)


   http.Handle("/metrics", promhttp.Handler())
   http.ListenAndServe(":2112", nil)
}


type Node struct {
   CpuTempDesc    *prometheus.Desc
   HdFailuresDesc *prometheus.Desc
}


func NewNode() *Node {
   return &Node{
      CpuTempDesc:    cpuTempDesc,
      HdFailuresDesc: hdFailuresDesc,
   }
}


// Describe implements prometheus.Collector.
func (e *Node) Describe(ch chan<- *prometheus.Desc) {
   ch <- e.CpuTempDesc
   ch <- e.HdFailuresDesc
}


// Collect implements prometheus.Collector.
func (e *Node) Collect(ch chan<- prometheus.Metric) {
   ch <- prometheus.MustNewConstMetric(cpuTempDesc, prometheus.CounterValue, 65.3)
   ch <- prometheus.MustNewConstMetric(hdFailuresDesc, prometheus.CounterValue, 1, "/dev/sda")
}
```
我们来看看func MustNewConstMetric(desc *Desc, valueType ValueType, value float64, labelValues ...string) Metric
```go
// MustNewConstMetric is a version of NewConstMetric that panics where
// NewConstMetric would have returned an error.
func MustNewConstMetric(desc *Desc, valueType ValueType, value float64, labelValues ...string) Metric {
   m, err := NewConstMetric(desc, valueType, value, labelValues...)
   if err != nil {
      panic(err)
   }
   return m
}

// NewConstMetric returns a metric with one fixed value that cannot be
// changed. Users of this package will not have much use for it in regular
// operations. However, when implementing custom Collectors, it is useful as a
// throw-away metric that is generated on the fly to send it to Prometheus in
// the Collect method. NewConstMetric returns an error if the length of
// labelValues is not consistent with the variable labels in Desc or if Desc is
// invalid.
func NewConstMetric(desc *Desc, valueType ValueType, value float64, labelValues ...string) (Metric, error) {
   if desc.err != nil {
      return nil, desc.err
   }
   if err := validateLabelValues(labelValues, len(desc.variableLabels)); err != nil {
      return nil, err
   }
   return &constMetric{
      desc:       desc,
      valType:    valueType,
      val:        value,
      labelPairs: MakeLabelPairs(desc, labelValues),
   }, nil
}
```

通过改方法的注释，NewConstMetric返回一个具有一个不能更改的固定值的度量。此软件包的用户在常规操作中不会有太多用处。但是，在实现自定义收集器时，它是一个很有用的一次性指标，可以在Collect方法中动态生成并发送给Prometheus。如果labelValues的长度与Desc中的变量标签不一致或Desc无效，NewConstMetric将返回错误。

[暴露指标](ExportMetric.md)