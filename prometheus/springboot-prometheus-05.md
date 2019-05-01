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



