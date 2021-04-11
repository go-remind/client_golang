Prometheus 的客户端库中提供了四种核心的指标类型：
* Counter（计数器）

Counter 类型代表一种样本数据单调递增的指标，即只增不减，除非监控系统发生了重置。例如，你可以使用 counter 类型的指标来表示服务的请求数、已完成的任务数、错误发生的次数等。

* Gauge（仪表盘）

Gauge 类型代表一种样本数据可以任意变化的指标，即可增可减。Gauge 通常用于像温度或者内存使用率这种指标数据，也可以表示能随时增加或减少的“总数”，例如：当前并发请求的数量。

* Histogram（直方图）

在大多数情况下人们都倾向于使用某些量化指标的平均值，例如 CPU 的平均使用率、页面的平均响应时间。这种方式的问题很明显，以系统 API 调用的平均响应时间为例：如果大多数 API 请求都维持在 100ms 的响应时间范围内，而个别请求的响应时间需要 5s，那么就会导致某些 WEB 页面的响应时间落到中位数的情况，而这种现象被称为长尾问题。
为了区分是平均的慢还是长尾的慢，最简单的方式就是按照请求延迟的范围进行分组。例如，统计延迟在 0~10ms 之间的请求数有多少而 10~20ms 之间的请求数又有多少。通过这种方式可以快速分析系统慢的原因。Histogram 和 Summary 都是为了能够解决这样问题的存在，通过 Histogram 和 Summary 类型的监控指标，我们可以快速了解监控样本的分布情况。
Histogram 在一段时间范围内对数据进行采样（通常是请求持续时间或响应大小等），并将其计入可配置的存储桶（bucket）中，后续可通过指定区间筛选样本，也可以统计样本总数，最后一般将数据展示为直方图。

* Summary（摘要）

与 Histogram 类型类似，用于表示一段时间内的数据采样结果（通常是请求持续时间或响应大小等），但它直接存储了分位数（通过客户端计算，然后展示出来），而不是通过区间来计算。
每种类型都有对应的vector版本：GaugeVec, CounterVec, SummaryVec, HistogramVec，vector版本细化了prometheus数据模型，增加了label(name=value)维度

以Gauge为例
```go
type Gauge interface {
    // 实现的接口
    Metric
    Collector
 
    // 设值
    Set(float64)
    // 增1
    Inc()
    // 减1
    Dec()
    // 加上一个值
    Add(float64)
    // 减去一个值
    Sub(float64)
}
```
要创建一个Gauge，可以调用
```go
func NewGauge(opts GaugeOpts) Gauge {

    // func newValue(desc *Desc, valueType ValueType, val float64, labelValues ...string) *value
    return newValue(NewDesc(

        BuildFQName(opts.Namespace, opts.Subsystem, opts.Name),

        opts.Help,
        nil,
        opts.ConstLabels,
    ), GaugeValue, 0)

}
```
其中每种基础类型都实现了Metric和Collector接口

Metric是所有指标的通用接口，表示需要导出到Prometheus的单个采样值+关联的元数据集 。此接口的实现包括Gauge、Counter、Histogram、Summary

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
Collector是任何Prometheus用来收集指标的对象，都需要实现此接口
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
收集器必须被注册（Registerer.Register）才能收集指标值。
内置的指标类型实现了此接口，包括GaugeVec、CounterVec、HistogramVec、SummaryVec

GaugeVec表示Gauge的向量
```go
type GaugeVec struct {

    *MetricVec
}
```
MetricVec表示用于bundle全限定名称、标签值有所不同的指标。通常不会直接使用此结构，而是将它作为具体指标向量GaugeVec, CounterVec, SummaryVec的一部分
```go
type MetricVec struct {

    mtx      sync.RWMutex // 保护元素的锁

    children map[uint64][]metricWithLabelValues // 所有指标实例（值+标签集）

    desc     *Desc // 描述符

 

    newMetric   func(labelValues ...string) Metric  // 以指定的标签值创建新指标

    hashAdd     func(h uint64, s string) uint64

    hashAddByte func(h uint64, b byte) uint64

}
```
opts创建大部分的指标类型时，可以通过该接口提供选项
```go
type Opts struct {
    // Namespace, Subsystem, Name是指标的全限定名称的组成部分，这些部分使用下划线连接
    // 仅仅Name是必须的
    Namespace string
    Subsystem string
    Name      string
 
    // 帮助信息，单个全限定名称，其帮助信息必须一样
    Help string
 
    // 常量标签用于为指标提供固定的标签，单个全限定名称，其常量标签集的所包含的标签名必须一致
    // 注意在大部分情况下，标签的值会变化，这些标签通常由指标矢量收集器（metric vector collector）来
    // 处理，例如CounterVec、GaugeVec、UntypedVec，而ConstLabels则仅用于特殊情况，例如：
    // vector collector (like CounterVec, GaugeVec, UntypedVec). ConstLabels
    // 1、在整个处理过程中，标签的值绝不会改变。这种标签例如运行中的二进制程序的修订版号
    // 2、在具有多个收集器（collector）来收集相同全限定名称的指标的情况下，那么每个收集器收集的
    //    指标的常量标签的值必须有所不同
    // 如果任何情况下，标签的值都不会改变，它可能更适合编码到全限定名称中
    ConstLabels Labels
}
```
**注意**：Gauge, Counter, Summary, Histogram自身就是接口，而GaugeVec, CounterVec, SummaryVec, HistogramVec, 和UntypedVec则不是接口。

下面我们来定义了两个指标数据，一个是Guage类型， 一个是CounterVec类型。分别代表了CPU温度和磁盘失败次数统计，使用上面的定义进行分类。
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




