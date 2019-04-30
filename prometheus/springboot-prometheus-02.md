# springboot整合prometheus(二)

springboot整合prometheus有两种方式：

1. 通过官方提供的依赖包`simpleclient_spring_boot`,但是好像对于`springboot2.X`不是很给力，对于`springboot2.X`我们可以使用一个桥接包`compile group: 'io.micrometer', name: 'micrometer-registry-prometheus', version: '1.0.6'`

2. 其实官方给的整合包`simpleclient_spring_boot`也是在自己原生方式上作了一个封装，方便大家使用，但是由于更新不即使，带来了使用上的不够灵活，因此下面先体验一下原生的方式


整个过程大致可以分为三个步骤：

**①收集Collector → ②注册Register → ③暴露Exporting**

## 1. Exporting

官方提供的暴露方式有三种：`http`、`Bridges`、`Pushgateway`

我们这里使用http的方式暴露，使用官方给我们提供好的`servlet`,因此需要因为官方的`servlet`,并且需要把这个`servlet`注册到springboot应用上下文中

### 添加依赖
```xml
<dependency><!--prometheus的java客户端-->
	<groupId>io.prometheus</groupId>
	<artifactId>simpleclient</artifactId>
	<version>0.5.0</version>
</dependency>
<dependency><!--通过这个servlet通过http的方式暴露出去-->
	<groupId>io.prometheus</groupId>
	<artifactId>simpleclient_servlet</artifactId>
	<version>0.5.0</version>
</dependency>
```

### 把这个servlet注册到springboot的应用上下文当中

```java
@SpringBootApplication
public class PrometheusDemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(PrometheusDemoApplication.class, args);
	}

	@Bean
	public ServletRegistrationBean<MetricsServlet> getServletRegistrationBean() {
		ServletRegistrationBean<MetricsServlet> bean = new ServletRegistrationBean<MetricsServlet>(new MetricsServlet());
		bean.addUrlMappings("/metrics");
		return bean;
	}

}
```

### 测试

请求地址`$ curl http://localhost:8080/metrics`，没有报错，说明成功，但是什么也没有返回

接下来开始收集数据并注册数据



## 2. Registering Metrics

### 注册

最好的方式注册一个`Metrics`是通过`static final`修饰的类成员变量，这样做可以防止同名的`Metrics`，我们也可以认为注册是定义一个指标的`TYPE`和`HTLP`，


```java
package com.example.prometheusdemo.demo;

import javax.annotation.PostConstruct;

import org.springframework.stereotype.Component;

import io.prometheus.client.Counter;

@Component
public class CounterDemo {
	
	
	//注册数据
	static final Counter REQUESTS = Counter.build().name("requests_total")
			.help("Total requests.").labelNames("label").register();
	
	
	//初始化数据方法一
	{
		REQUESTS.labels("trace").inc(0D);
	}
	
	
	//初始化数据方法二
	@PostConstruct
	public void processRequest() {
		 
		    REQUESTS.labels("error").inc();
	}
	
	
}

```

### 验证是否成功

```bash
$ curl http://localhost:8080/metrics
# HELP requests_total Total requests.
# TYPE requests_total counter
requests_total{label="trace",} 0.0
requests_total{label="error",} 1.0
```

## 3. Collectors

这里的收集器就是数据收集器

### 官方提供的Collectors

1. 添加依赖
2. 开启暴露

#### 1. jvm

Java客户端包括用于垃圾收集，内存池，JMX，类加载和线程计数的收集器。这些可以单独添加，也可以使用DefaultExports方便地注册它们。

监控虚拟机相关的指标项：



```xml
<dependency>
	<groupId>io.prometheus</groupId>
	<artifactId>simpleclient_hotspot</artifactId>
	<version>0.5.0</version>
</dependency>
```

```java
import javax.annotation.PostConstruct;

import org.springframework.stereotype.Component;

import io.prometheus.client.hotspot.DefaultExports;
import lombok.extern.slf4j.Slf4j;

@Component
@Slf4j
public class PrometheusConfig {


    @PostConstruct
    public void initialize() {
        log.info("prometheus init...");
       DefaultExports.initialize();
        log.info("prometheus has been initialized...");
    }

}

```

#### 2. log

支持的log有`log4j`、`log4j2`和`logback`

我们以`logback`为例

```xml
<dependency>
	<groupId>io.prometheus</groupId>
	<artifactId>simpleclient_logback</artifactId>
	<version>0.5.0</version>
</dependency>

```
logback-spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml"/>

    <appender name="METRICS" class="io.prometheus.client.logback.InstrumentedAppender" />

    <root level="INFO">
        <appender-ref ref="METRICS" />
    </root>

</configuration>

```

效果如下：

```
# HELP logback_appender_total Logback log statements at various log levels
# TYPE logback_appender_total counter
logback_appender_total{level="debug",} 0.0
logback_appender_total{level="warn",} 0.0
logback_appender_total{level="trace",} 0.0
logback_appender_total{level="error",} 1.0
logback_appender_total{level="info",} 15.0
```

#### 3. 其他

1. Guava cache
2. Hibernate SessionFactory
3. Jetty server metrics




### 自定义Collectors

有时无法直接检测代码，因为它不在您的控制范围内。这要求您从其他系统代理指标

继承`Collector`实现`collect`方法

```java
public class YourCustomCollector extends Collector{

	//每次都会执行这个方法
	@Override
	public List<MetricFamilySamples> collect() {
	    List<MetricFamilySamples> mfs = new ArrayList<MetricFamilySamples>();
	    // With no labels.
	    mfs.add(new GaugeMetricFamily("my_gauge", "help", 42));
	    // With labels
	    GaugeMetricFamily labeledGauge = new GaugeMetricFamily("my_other_gauge", "help", Arrays.asList("labelname"));
	    labeledGauge.addMetric(Arrays.asList("foo"), 4);
	    labeledGauge.addMetric(Arrays.asList("bar"), 5);
	    mfs.add(labeledGauge);
	    return mfs;
	  }

}
```

注册到成员变量上

```java


import org.springframework.stereotype.Component;

import com.example.prometheusdemo.config.YourCustomCollector;

import io.prometheus.client.Gauge;
@Component
public class GaugeDemo {


	
	static final YourCustomCollector requests = new YourCustomCollector().register();

}
```

> SummaryMetricFamily的工作方式类似





