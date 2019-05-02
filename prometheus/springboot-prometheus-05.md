# springboot整合prometheus(五)

![](/assets/20190502002122.png)

---

![](/assets/20190502191124.png)


##　Prometheus 的特点

* 多维数据模型（时序列数据由metric名和一组key/value组成）
* 在多维度上灵活的查询语言(PromQl)
* 不依赖分布式存储，单主节点工作.
* 通过基于HTTP的pull方式采集时序数据
* 可以通过中间网关进行时序列数据推送(pushing)
* 目标服务器可以通过发现服务或者静态配置实现
* 多种可视化和仪表盘支持

## TSDB是什么？ (Time Series Database)

简单的理解为.一个优化后用来处理时间序列数据的软件,并且数据中的数组是由时间进行索引的

* 大部分时间都是写入操作
* 写入操作几乎是顺序添加;大多数时候数据到达后都以时间排序.
* 写操作很少写入很久之前的数据,也很少更新数据.大多数情况在数据被采集到数秒或者数分钟后就会被写入数据库.
* 删除操作一般为区块删除,选定开始的历史时间并指定后续的区块.很少单独删除某个时间或者分开的随机时间的数据.
* 数据一般远远超过内存大小,所以缓存基本无用.系统一般是 IO 密集型
* 读操作是十分典型的升序或者降序的顺序读,
* 高并发的读操作十分常见.


## 做一个案例，基于springboot整合prometheus监控http

Counter:只增不减的计数器，记录应用请求的总量
Gauge: 可增可减的仪表盘，我们使用Gauge记录当前正在处理的Http请求数量
Histogram: 分布统计图
Summary: 分布统计图

### gradle

```groovy
plugins {
	id 'org.springframework.boot' version '2.1.4.RELEASE'
	id 'java'
}

apply plugin: 'io.spring.dependency-management'

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

configurations {
	compileOnly {
		extendsFrom annotationProcessor
	}
}

repositories {
	mavenCentral()
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-actuator'
	implementation 'org.springframework.boot:spring-boot-starter-web'
	implementation 'io.micrometer:micrometer-core'
	implementation 'io.micrometer:micrometer-registry-prometheus'
	compileOnly 'org.projectlombok:lombok'
	runtimeOnly 'org.springframework.boot:spring-boot-devtools'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

```
### springboot配置

```properties
management.metrics.enable.all=false
management.metrics.export.prometheus.enabled=true
management.endpoint.metrics.enabled=true
management.endpoint.prometheus.enabled=true
management.endpoints.web.exposure.include=prometheus,health
```

### 1. 请求总量的监控Counter

最终的统计结果样式：

```
# HELP http_requests_total http请求总计数
# TYPE http_requests_total counter
http_requests_total{path="/hello",method="GET",code="200",} 3.0
http_requests_total{path="/world",method="GET",code="200",} 5.0
```

首先需要定义数据收集器collector

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

import io.prometheus.client.CollectorRegistry;
import io.prometheus.client.Counter;

@Component
public class CounterMetrics {

	@Autowired
	private CollectorRegistry collectorRegistry;

	@Bean
	public Counter httpRequestsTotalCounterCollector() {
		return Counter.build().name("http_requests_total").labelNames("path", "method", "code").help("http请求总计数")
				.register(collectorRegistry);
	}

}
```

定义一个http拦截器

```java
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import io.prometheus.client.Counter;
import lombok.extern.slf4j.Slf4j;


@Slf4j
public class PrometheusInterceptor implements HandlerInterceptor {
	
	@Autowired
	@Qualifier("httpRequestsTotalCounterCollector")
	private Counter httpRequestsTotalCounterCollector;

	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		log.info("Prometheus intercrptor preHandle ...");

		String requestURI = request.getRequestURI();
        String method = request.getMethod();
        int status = response.getStatus();
		
        httpRequestsTotalCounterCollector.labels(requestURI, method, String.valueOf(status)).inc();

		return HandlerInterceptor.super.preHandle(request, response, handler);
	}

	@Override
	public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
			ModelAndView modelAndView) throws Exception {
		log.info("Prometheus intercrptor postHandle ...");
		HandlerInterceptor.super.postHandle(request, response, handler, modelAndView);
	}

	@Override
	public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
			throws Exception {
		log.info("Prometheus intercrptor afterCompletion ...");
		
		
		
		HandlerInterceptor.super.afterCompletion(request, response, handler, ex);
	}
	
	

}
```

注册http拦截器使其生效

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import com.example.demo.interceptor.PrometheusInterceptor;

@Configuration
public class InterceptorConfiguration implements WebMvcConfigurer {


	@Bean
	public HandlerInterceptor prometheusInterceptor() {
		return new PrometheusInterceptor();
	}

	@Override
	public void addInterceptors(InterceptorRegistry registry) {
		registry.addInterceptor(prometheusInterceptor()).addPathPatterns("/**");
	}
}

```

定义两个http请求接口

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {
	
	
	@GetMapping("hello")
	public Object sayHello() {
		return "hello prometheus";
	}
	
	@GetMapping("world")
	public Object sayWorld() {
		return "world prometheus";
	}


}
```


> 1. 使用Counter.build()创建Counter metrics，
> 2. name()方法，用于指定该指标的名称 
> 3. labelNames()方法，用于声明该metrics拥有的维度label。
> 4. 在preHandle方法中，我们获取当前请求的，RequesPath，Method以及状态码。并且调用inc()方法，在每次请求发生时计数+1。
> 5. Counter.build()…register(),会像Collector中注册该指标，并且当访问/metrics地址时，返回该指标的状态。

通过指标http_requests_total我们可以：

- 查询应用的请求总量

```
# PromQL
sum(http_requests_total)
```

- 查询每秒Http请求量

```
# PromQL
sum(rate(http_requests_total[5m]))
```

- 查询当前应用请求量Top N的URI

```
# PromQL
topk(10, sum(http_requests_total) by (path))
```

### 2. 当前正在处理的http请求数


最后的显示样式是：

```
# HELP http_inprogress_requests http当前正在处理的请求数
# TYPE http_inprogress_requests gauge
http_inprogress_requests{path="/hello",method="GET",code="200",} 1.0
# HELP http_requests_total http请求总计数
# TYPE http_requests_total counter
http_requests_total{path="/hello",method="GET",code="200",} 3.0
```

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

import io.prometheus.client.CollectorRegistry;
import io.prometheus.client.Gauge;

@Component
public class GuageMetrics {
	@Autowired
	private CollectorRegistry collectorRegistry;
	
	
	@Bean
	public Gauge httpInprogressRequestsGuageCollector() {
		
		return Gauge.build()
	            .name("http_inprogress_requests").labelNames("path", "method", "code")
	            .help("http当前正在处理的请求数").register(collectorRegistry);
		
	}
}
```

```java
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import io.prometheus.client.Counter;
import io.prometheus.client.Gauge;
import lombok.extern.slf4j.Slf4j;


@Slf4j
public class PrometheusInterceptor implements HandlerInterceptor {
	
	//http请求总数
	@Autowired
	@Qualifier("httpRequestsTotalCounterCollector")
	private Counter httpRequestsTotalCounterCollector;
	//当前正在处理的请求数
	@Autowired
	@Qualifier("httpInprogressRequestsGuageCollector")
	private Gauge httpInprogressRequestsGuageCollector;

	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		log.info("Prometheus intercrptor preHandle ...");
		String requestURI = request.getRequestURI();
        String method = request.getMethod();
        int status = response.getStatus();
		
        httpRequestsTotalCounterCollector.labels(requestURI, method, String.valueOf(status)).inc();
        
        //当前正在处理的请求数 + 1
        httpInprogressRequestsGuageCollector.labels(requestURI, method, String.valueOf(status)).inc();
		return HandlerInterceptor.super.preHandle(request, response, handler);
	}

	@Override
	public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
			ModelAndView modelAndView) throws Exception {
		log.info("Prometheus intercrptor postHandle ...");
		HandlerInterceptor.super.postHandle(request, response, handler, modelAndView);
	}

	@Override
	public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
			throws Exception {
		log.info("Prometheus intercrptor afterCompletion ...");
		
		String requestURI = request.getRequestURI();
        String method = request.getMethod();
        int status = response.getStatus();
		//当前正在处理的请求数 - 1
        httpInprogressRequestsGuageCollector.labels(requestURI, method, String.valueOf(status)).dec();
		HandlerInterceptor.super.afterCompletion(request, response, handler, ex);
	}
	
	

}

```

### 3. Histogram：自带buckets区间用于统计分布统计图

http请求内容长度的区间分布图

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

import io.prometheus.client.CollectorRegistry;
import io.prometheus.client.Histogram;


@Component
public class HistogramMetrics {
	
	
	@Autowired
	private CollectorRegistry collectorRegistry;
	
	

	
	@Bean
	public Histogram httpRequestsBytesHistogramCollector() {
		
		return Histogram.build().labelNames("path", "method", "code")
        .name("http_requests_bytes_histogram").help("http bucket 请求大小区间分布图")
        .register(collectorRegistry);
		
	}

}
```

```java
package com.example.demo.interceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import io.prometheus.client.Counter;
import io.prometheus.client.Gauge;
import io.prometheus.client.Histogram;
import lombok.extern.slf4j.Slf4j;


@Slf4j
public class PrometheusInterceptor implements HandlerInterceptor {
	

	@Autowired
	@Qualifier("httpRequestsBytesHistogramCollector")
	private Histogram httpRequestsBytesHistogramCollector;

	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		log.info("Prometheus intercrptor preHandle ...");
		String requestURI = request.getRequestURI();
        String method = request.getMethod();
        int status = response.getStatus();
		
        
        //请求大小
        httpRequestsBytesHistogramCollector.labels(requestURI, method, String.valueOf(status)).observe(request.getContentLength());
        
        
		return HandlerInterceptor.super.preHandle(request, response, handler);
	}

	
	

}

```
我们可以看到，自己通过`le`标签划分了区间，默认的划分间隔就是下面这样的0-10

```
# HELP http_requests_bytes_histogram http bucket 请求大小区间分布图
# TYPE http_requests_bytes_histogram histogram
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="0.005",} 2.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="0.01",} 2.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="0.025",} 2.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="0.05",} 2.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="0.075",} 2.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="0.1",} 2.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="0.25",} 2.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="0.5",} 2.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="0.75",} 2.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="1.0",} 2.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="2.5",} 2.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="5.0",} 2.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="7.5",} 2.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="10.0",} 2.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="+Inf",} 2.0
http_requests_bytes_histogram_count{path="/hello",method="GET",code="200",} 2.0
http_requests_bytes_histogram_sum{path="/hello",method="GET",code="200",} -2.0
```

如果value值在0-100之间的数值，我们就不得不自己划分区间
使用Histogram构造器可以创建Histogram监控指标。默认的buckets范围为{.005, .01, .025, .05, .075, .1, .25, .5, .75, 1, 2.5, 5, 7.5, 10}。如果需要覆盖默认的buckets，可以使用.buckets(double… buckets)覆盖。

```java
	@Bean
	public Histogram httpRequestsBytesHistogramCollector() {
		
		return Histogram.build().labelNames("path", "method", "code")
        .name("http_requests_bytes_histogram").help("http bucket 请求大小区间分布图")
        .buckets(30,60,90, 100)
        .register(collectorRegistry);
		
	}
```

```
# HELP http_requests_bytes_histogram http bucket 请求大小区间分布图
# TYPE http_requests_bytes_histogram histogram
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="30.0",} 6.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="60.0",} 9.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="90.0",} 10.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="100.0",} 11.0
http_requests_bytes_histogram_bucket{path="/hello",method="GET",code="200",le="+Inf",} 11.0
http_requests_bytes_histogram_count{path="/hello",method="GET",code="200",} 11.0
http_requests_bytes_histogram_sum{path="/hello",method="GET",code="200",} 427.0
```

### 4. Summary

```java
Summary.build()
        .name("http_requests_bytes_summary")
        .labelNames("path", "method", "code")
        .help("Request bytes ").register(collectorRegistry);

```

```java
httpRequestsBytesSummaryCollector.labels(requestURI, method, String.valueOf(status)).observe(new Random().nextInt(100));
```

```
# HELP http_requests_bytes_summary Request bytes 
# TYPE http_requests_bytes_summary summary
http_requests_bytes_summary_count{path="/hello",method="GET",code="200",} 6.0
http_requests_bytes_summary_sum{path="/hello",method="GET",code="200",} 233.0
```

常用的百分位数为0.5-quantile，0.9-quantile以及0.99-quantile。这也是Prometheus默认的设置。

这只是Prometheus中Summary目前版本的默认设置，在版本v0.10中，这些默认值会废弃，意味着默认的Summary将没有quantile设置。

自定义百分比区间

```java
Summary.build()
        .name("http_requests_bytes_summary")
        .quantile(0.5, 0.05)
        .quantile(0.9, 0.01)
        .labelNames("path", "method", "code")
        .help("Request bytes ").register(collectorRegistry);
```
对面下面这一系列值的统计结果是：

> 6，14，15，23，24，51，79，86，92，95

```
# HELP http_requests_bytes_summary Request bytes 
# TYPE http_requests_bytes_summary summary
http_requests_bytes_summary{path="/hello",method="GET",code="200",quantile="0.5",} 24.0
http_requests_bytes_summary{path="/hello",method="GET",code="200",quantile="0.9",} 92.0
http_requests_bytes_summary_count{path="/hello",method="GET",code="200",} 10.0
http_requests_bytes_summary_sum{path="/hello",method="GET",code="200",} 485.0
```


### 5. 自定义collector

```java
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Random;

import io.prometheus.client.Collector;
import io.prometheus.client.GaugeMetricFamily;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class MyCollector extends Collector{

	@Override
    public List<MetricFamilySamples> collect() {
		
		log.info("============custom defined collector===================");
        List<MetricFamilySamples> mfs = new ArrayList<MetricFamilySamples>();
        // With no labels.
        mfs.add(new GaugeMetricFamily("my_gauge", "help", 42));
        // With labels
        GaugeMetricFamily labeledGauge = new GaugeMetricFamily("my_other_gauge", "help", Arrays.asList("labelname"));
        labeledGauge.addMetric(Arrays.asList("foo"), new Random().nextInt(5));
        labeledGauge.addMetric(Arrays.asList("bar"), new Random().nextInt(5));
        mfs.add(labeledGauge);
        return mfs;
      }


}

```

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

import io.prometheus.client.CollectorRegistry;

@Component
public class MyCollectorMetrics {
	
	@Autowired
	private CollectorRegistry collectorRegistry;

	@Bean
	public MyCollector myCollector() {
		return new MyCollector().register(collectorRegistry);
	}

}
```

prometheus每次拉取metrics的时候，都会执行`MyCollector.collect()`方法

prometheus提供的collector和自己定义的collector区别：

prometheus定义的是成员变量，每次prometheus拉取数据的时候，直接读取内存中的数据，内存中的数据是我们在数据改变的时候就设置的到内存中了已近，说白了就是我们把数据推到内存中的成员变量上，
而prometheus直接从成员变量上拉取数据
自定义的collector是每次prometheus读取数据的时候，都要执行方法，从数据哭或者其他方法获取数据,这些数据都是计算好了的数据，例如counter可以做累计，我们只需要每次inc()+1，成员变量标签值上会帮助我们累计值，但是如果使用自己定义的counter,我们设置的值就是直接当时的值，可以理解成自己定义的collector是计算好的当时的结果值，直接存入到数据库中，而我们每次直接从数据库中读取当时的值


### 6. Timer计时器

Timer(计时器)同时测量一个特定的代码逻辑块的调用(执行)速度和它的时间分布。简单来说，就是在调用结束的时间点记录整个调用块执行的总时间，适用于测量短时间执行的事件的耗时分布

* 适用于label的value是一个时间长度，用于时间的分布统计
* 支持Histogram和Summary

```java
//Histogram支持计时器
class YourClass {
  static final Histogram requestLatency = Histogram.build()
     .name("requests_latency_seconds").help("Request latency in seconds.").register();

  void processRequest(Request req) {
    Histogram.Timer requestTimer = requestLatency.startTimer();//开始计时
    try {
      // Your code here.
    } finally {
      requestTimer.observeDuration();//计时结束
    }
  }
}
//Summary也支持计时器
class YourClass {
  static final Summary receivedBytes = Summary.build()
     .name("requests_size_bytes").help("Request size in bytes.").register();
  static final Summary requestLatency = Summary.build()
     .name("requests_latency_seconds").help("Request latency in seconds.").register();

  void processRequest(Request req) {
    Summary.Timer requestTimer = requestLatency.startTimer();
    try {
      // Your code here.
    } finally {
      receivedBytes.observe(req.size());
      requestTimer.observeDuration();
    }
  }
}
```

还有一种写法

```java
class YourClass {
  static final Histogram requestLatency = Histogram.build()
     .name("requests_latency_seconds").help("Request latency in seconds.").register();

  void processRequest(Request req) {
    requestLatency.time(new Runnable() {
      public abstract void run() {
        // Your code here.
      }
    });


    // Or the Java 8 lambda equivalent
    requestLatency.time(() -> {
      // Your code here.
    });
  }
}
class YourClass {
  static final Summary requestLatency = Summary.build()
    .quantile(0.5, 0.05)   // Add 50th percentile (= median) with 5% tolerated error
    .quantile(0.9, 0.01)   // Add 90th percentile with 1% tolerated error
    .name("requests_latency_seconds").help("Request latency in seconds.").register();

  void processRequest(Request req) {
    requestLatency.time(new Runnable() {
      public abstract void run() {
        // Your code here.
      }
    });


    // Or the Java 8 lambda equivalent
    requestLatency.time(() -> {
      // Your code here.
    });
  }
}
```

http请求时长的统计


























