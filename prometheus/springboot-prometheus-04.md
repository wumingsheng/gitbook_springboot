# springboot整合prometheus(四)

![](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1556722981906&di=4aa1755b5f528609311908f95f79fabf&imgtype=0&src=http%3A%2F%2Fwww.maxon.net%2Fuploads%2Fpics%2Fweb_p54.jpg)              


prometheus是通过拉取的方式获取数据，prometheus是时序数据库，是用时间为索引，每个label监控项上是一些key-value的数据，数据是累计存储到数据库中的

prometheus拉取springboot上的监控数据的时候，springboot的监控数据是存储在项目内部基于当时时间点的数据，so springboot内部存储的数据只是一个时间点的数据（当时），
因此不用担心springboot项目会占用太多的内存，监控项是固定的，只是随时间数据在不断的变化，而peometheus会拉取每一个时间的数据，存储在时序数据库中，作分析统计

## 如何区分prometheus中Histogram和Summary类型的metrics？

Histogram和Summary主用用于统计和分析样本的分布情况。

在大多数情况下人们都倾向于使用某些量化指标的平均值，例如CPU的平均使用率、页面的平均响应时间。这种方式的问题很明显，以系统API调用的平均响应时间为例：如果大多数API请求都维持在100ms的响应时间范围内，而个别请求的响应时间需要5s，那么就会导致某些WEB页面的响应时间落到中位数的情况，而这种现象被称为长尾问题。

为了区分是平均的慢还是长尾的慢，最简单的方式就是按照请求延迟的范围进行分组。例如，统计延迟在0~10ms之间的请求数有多少而10~20ms之间的请求数又有多少。通过这种方式可以快速分析系统慢的原因。Histogram和Summary都是为了能够解决这样问题的存在，通过Histogram和Summary类型的监控指标，我们可以快速了解监控样本的分布情况。


例如，指标prometheus_tsdb_wal_fsync_duration_seconds的指标类型为Summary。 它记录了Prometheus Server中wal_fsync处理的处理时间，通过访问Prometheus Server的/metrics地址，可以获取到以下监控样本数据：

```
# HELP prometheus_tsdb_wal_fsync_duration_seconds Duration of WAL fsync.
# TYPE prometheus_tsdb_wal_fsync_duration_seconds summary
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.5"} 0.012352463
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.9"} 0.014458005
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.99"} 0.017316173
prometheus_tsdb_wal_fsync_duration_seconds_sum 2.888716127000002
prometheus_tsdb_wal_fsync_duration_seconds_count 216
```

从上面的样本中可以得知当前Prometheus Server进行wal_fsync操作的总次数为216次，耗时2.888716127000002s。其中中位数（quantile=0.5）的耗时为0.012352463，9分位数（quantile=0.9）的耗时为0.014458005s。

在Prometheus Server自身返回的样本数据中，我们还能找到类型为Histogram的监控指标prometheus_tsdb_compaction_chunk_range_bucket。

```
# HELP prometheus_tsdb_compaction_chunk_range Final time range of chunks on their first compaction
# TYPE prometheus_tsdb_compaction_chunk_range histogram
prometheus_tsdb_compaction_chunk_range_bucket{le="100"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="400"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="1600"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="6400"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="25600"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="102400"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="409600"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="1.6384e+06"} 260
prometheus_tsdb_compaction_chunk_range_bucket{le="6.5536e+06"} 780
prometheus_tsdb_compaction_chunk_range_bucket{le="2.62144e+07"} 780
prometheus_tsdb_compaction_chunk_range_bucket{le="+Inf"} 780
prometheus_tsdb_compaction_chunk_range_sum 1.1540798e+09
prometheus_tsdb_compaction_chunk_range_count 780
```

与Summary类型的指标相似之处在于Histogram类型的样本同样会反应当前指标的记录的总数(以_count作为后缀)以及其值的总量（以_sum作为后缀）。不同在于Histogram指标直接反应了在不同区间内样本的个数，区间通过标签len进行定义。

同时对于Histogram的指标，我们还可以通过`histogram_quantile()`函数计算出其值的分位数。不同在于Histogram通过histogram_quantile函数是在服务器端计算的分位数。 而Sumamry的分位数则是直接在客户端计算完成。因此对于分位数的计算而言，Summary在通过PromQL进行查询时有更好的性能表现，而Histogram则会消耗更多的资源。反之对于客户端而言Histogram消耗的资源更少。在选择这两种方式时用户应该按照自己的实际场景进行选择。


## 百分位数（quantile）

Prometheus中称为quantile，其实叫percentile更准确。百分位数是指小于某个特定数值的采样点达到一定的百分比。例如，假设0.9-quantile的值为120，意思就是所有的采样值中，小于120的采样值的数量占总体采样值的90%。相应的，假设0.5-quantile的值为x，那么意思就是指小于x的采样值占总体的50%，所以0.5-quantile也指中值（median）。

相对于简单的平均值来说，百分位数更丰富，更能反应出真实的用户体验。常用的百分位数为0.5-quantile，0.9-quantile以及0.99-quantile。这也是Prometheus默认的设置。

> 这只是Prometheus中Summary目前版本的默认设置，在版本v0.10中，这些默认值会废弃，意味着默认的Summary将没有quantile设置。


## bucket

Histogram主要是设置不同的bucket，采用值分别落入不同的bucket。例如上面第一个bucket就是响应时间小于10ms的采样点的数量，第二个bucket就是响应时间小于50ms的采样点的数量，依此类推。

注意后面的采样点是包含前面的采用点的，例如xxx_bucket的值为30，而xxx_bucket的值为120，那么意味着这120个采用点中，有30个是小于10ms的，其余90个采样点的响应时间是介于10ms和50ms之间的。
注意+Inf是最高bucket的上限值，所以xxx_bucket是所有采样点的数量，是Prometheus自动增加的一个bucket。


## 区别

1. histogram有bucket(区间)，summary有quatile(分位数|百分比)

2. summary分位数是客户端(springboot项目)计算上报，histogram中位数是服务端（prometheus server）计算（通过promQL内置函数）

	> 计算quantile值直接用函数histogram_quantile即可，例如下面是计算0.9-quantile的值，
	> histogram_quantile(0.9, rate(http_request_duration_milliseconds_bucket[10m]))

3. Summary不能对quantile值进行aggregation操作，而Histogram则可以；所以如果针对多实例的场景计算quantile，只能使用Histogram

	> 使用Summary要注意一点，就是不能对Summary产生的quantile值进行aggregation运算（例如sum, avg等）。例如有两个实例同时运行，都对外提供服务，分别统计各自的响应时间。最后分别计算出的0.5-quantile的值为60和80，这时如果简单的求平均(60+80)/2，认为是总体的0.5-quantile值，那么就错了。如果你闭上眼睛，简单思考一下，就会明白对两个quantile值求平均毫无意义。所以如果需要对多个实例的quantile值进行aggregation操作，那么就不能使用Summary。

4. 如果histogram的bucket设置不合理，则最后误差可能会很大；所以如果需要相对精确的结果，而且是单实例场景，那么就使用Summary

	> 使用Histogram计算quantile值，最大的问题就是：因为Histogram采用了线性插值法，所以如果bucket设置不合理，那么最后计算出的值可能偏差比较大。例如在前面的例子中，假设0.9-quantile的结果在10ms~50ms之间，但是表达式必须返回一个具体的值，这时就采用线性插值法得出36ms。显然这种方法计算出的值可能会有误差，而且范围越大，例如10ms ~ 500ms，那么误差也会越大。
	> 50-10）*0.9=36ms


5. Summary计算出的quantile值是基于进程开始运行至今的所有采样值计算出来的；而Histogram则是基于最近的一段时间的采样值计算出来的，更符合monitoring系统的本质。

使用Histogram计算quantile值，最大的问题就是：因为Histogram采用了线性插值法，所以如果bucket设置不合理，那么最后计算出的值可能偏差比较大。例如在前面的例子中，假设0.9-quantile的结果在10ms~50ms之间，但是表达式必须返回一个具体的值，这时就采用线性插值法得出36ms。显然这种方法计算出的值可能会有误差，而且范围越大，例如10ms ~ 500ms，那么误差也会越大。









