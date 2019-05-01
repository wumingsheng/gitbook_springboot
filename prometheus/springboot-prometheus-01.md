# springboot整合prometheus(一)

## 1、Counter

计数器是表示单个单调递增计数器的累积量，其值只能增加或在重启时重置为零。 

1. 服务的请求数量
2. 已完成的任务数
3. 出现的错误总数

> 不要使用计数器来监控可能减少的值。 例如，不要使用计数器来处理当前正在运行的进程数，而应该用Gauge。

## 2、Gauge

Gauge可以用来存放一个可以任意变大变小的数值，通常用于测量值Gauge可以任意加减

1. 温度
2. 内存
3. cpu
4. 线程数



## 3、Histogram

直方图，内置分析样本的分布情况

1. 请求持续时间
2. 响应大小。
3. 区间数据分组统计

## 4、Summary

摘要 自定义样本分布情况

1. 观察总和
2. 观察计数 
3. 排名估计

> 典型的用例是观察请求延迟。 默认情况下，Summary提供延迟的中位数。


> 1. [https://blog.csdn.net/michaelgo/article/details/81709652](https://blog.csdn.net/michaelgo/article/details/81709652)
> 2. [https://github.com/prometheus/client_java](https://github.com/prometheus/client_java)
> 3. [https://www.evernote.com/client/web?login=true#?n=a0e24e9f-c550-4e6f-8ec3-2beb132f00e3&query=prometheus&s=s576&](https://www.evernote.com/client/web?login=true#?n=a0e24e9f-c550-4e6f-8ec3-2beb132f00e3&query=prometheus&s=s576&)






















