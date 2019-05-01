# springboot整合prometheus(三)

## 1. 加入prometheus依赖

prometheus 官方提供了spring boot 的依赖，但是该客户端已经不支持spring boot 2

```xml
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_spring_boot</artifactId>
    <version>0.4.0</version>
</dependency>
```

由于 spring boot 2 的actuator 使用了 Micrometer 进行监控数据统计，
而Micrometer 提供了prometheus 支持，我们可以使用 micrometer-registry-prometheus 来集成 spring boot 2
加入相应依赖

```xml
 <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
 <dependency>
        <groupId>io.micrometer</groupId>
         <artifactId>micrometer-core</artifactId>
 </dependency>
 <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
 </dependency>
```

## 2. 配置开启prometheus

```yaml
management:
  metrics:
    enable:
      all: false # all mean all inner metrics 
    export:
      prometheus:
        enabled: true
    tags: # user define labels only for inner metrics
      label-1-name: hello
      label-2-name: world
  endpoint:
    metrics:
      enabled: true
    prometheus:
      enabled: true
  endpoints:
    web:
      exposure:
        include: ["prometheus","health"]
```
## 3. 验证

```sh
$ curl localhost:8080/actuator/prometheus
```

## 4. 关闭不需要的内置监控项

Whether meter IDs starting-with the specified name should be enabled. The longest match wins, the key `all` can also be used to configure all meters.

```yaml
#服务停止
management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    shutdown:
      enabled: true
    restart:
      enabled: true
  metrics:
    enable:
      jvm: true
      logback: false
      process.files: false
      process.uptime: false
      process.start.time: false
      system.cpu: false
      process.cpu: false
      tomcat: false
      http: false
      system: false
```


## 4. 配置自定义监控项


```java
package com.example.prometheusdemo.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

import io.prometheus.client.CollectorRegistry;
import io.prometheus.client.Counter;


@Component
public class PromusConfig {
	

	@Autowired
	private CollectorRegistry collectorRegistry;
	
	
	@Bean
  public Counter requestTotalCountCollector(){
      return  Counter.build()
       .name("http_requests_total")
       .labelNames("path", "method", "code")
       .help("http请求总计数").register(collectorRegistry);
  }

}

```


```java

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import io.prometheus.client.Counter;

@RestController
public class DemoController {
	
	@Autowired
  @Qualifier("requestTotalCountCollector")
	private Counter counter;
	
	@GetMapping("ddd")
	public Object deoi() {
		counter.labels("path1", "method1", "code1").inc();
		return "ok";
	}

}

```

## 自定义一个metrics 收集器

https://segmentfault.com/a/1190000018642077

自定义一个metrics 收集器,只需要继承 prometheus 的 Collector，重写抽象方法collect

```java
package com.example.prometheusdemo.config;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Random;

import io.prometheus.client.Collector;
import io.prometheus.client.GaugeMetricFamily;



public class YourCustomCollector extends Collector{

	@Override
	public List<MetricFamilySamples> collect() {
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

注册 YourCustomCollector 到 spring 容器

```java

@Component
public class PromusConfig {
	

	@Autowired
	private CollectorRegistry collectorRegistry;
	
	
	@Bean
  public Counter requestTotalCountCollector(){
      return  Counter.build()
       .name("http_requests_total")
       .labelNames("path", "method", "code")
       .help("http请求总计数").register(collectorRegistry);
  }
	
	@Bean
  @Primary
  public YourCustomCollector yourCustomCollector(){
      return new YourCustomCollector().register(collectorRegistry);
  }

}
```











