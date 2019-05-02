# springboot整合prometheus(五)

![](/assets/20190502002122.png)


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

### 请求总量的监控

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
		
        httpRequestsTotalCounterCollector.labels(requestURI, method, String.valueOf(status)).inc();
		
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






