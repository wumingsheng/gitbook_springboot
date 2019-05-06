# 使用Sentinel实现接口限流

> https://mp.weixin.qq.com/s?__biz=MzAxODcyNjEzNQ==&mid=2247487191&idx=1&sn=a8feb651b9e7dc12a164b1b3b8a82fee&chksm=9bd0a34faca72a594cf8eacc384e40e3176bd2b518c9caa640d3afaa38d75472b9e5d66cfb4d&scene=4
> http://www.pianshen.com/article/1148196206/

Nacos作为注册中心和配置中心的基础教程，到这里先告一段落，后续与其他结合的内容等讲到的时候再一起拿出来说，不然内容会有点跳跃。接下来我们就来一起学习一下Spring Cloud Alibaba下的另外一个重要组件：Sentinel。

## 限流组件Sentinel

- Sentinel是把流量作为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性。
- 默认支持 Servlet、Feign、RestTemplate、Dubbo 和 RocketMQ 限流降级功能的接入，可以在运行时通过控制台实时修改限流降级规则，还支持查看限流降级 Metrics 监控。
- 自带控台动态修改限流策略。但是每次服务重启后就丢失了。所以它也支持ReadableDataSource 目前支持file, nacos, zk, apollo 这4种类型


## Sentinel是什么

Sentinel的官方标题是：分布式系统的流量防卫兵。从名字上来看，很容易就能猜到它是用来作服务稳定性保障的。对于服务稳定性保障组件，如果熟悉Spring Cloud的用户，第一反应应该就是Hystrix。但是比较可惜的是Netflix已经宣布对Hystrix停止更新。那么，在未来我们还有什么更好的选择呢？除了Spring Cloud官方推荐的resilience4j之外，目前Spring Cloud Alibaba下整合的Sentinel也是用户可以重点考察和选型的目标。

Sentinel的功能和细节比较多，一篇内容很难介绍完整。所以下面我会分多篇来一一介绍Sentinel的重要功能。本文就先从限流入手，说说如何把Sentinel整合到Spring Cloud应用中，以及如何使用Sentinel Dashboard来配置限流规则。通过这个简单的例子，先将这一套基础配置搭建起来。






## 使用Sentinel实现接口限流

Sentinel的使用分为两部分：

1. sentinel-dashboard：与hystrix-dashboard类似，但是它更为强大一些。除了与hystrix-dashboard一样提供实时监控之外，还提供了流控规则、熔断规则的在线维护等功能。
2. 客户端整合：每个微服务客户端都需要整合sentinel的客户端封装与配置，才能将监控信息上报给dashboard展示以及实时的更改限流或熔断规则等。

下面我们就分两部分来看看，如何使用Sentienl来实现接口限流。

### 部署Sentinel Dashboard

本文采用的spring cloud alibaba版本是0.2.1，可以查看依赖发现当前版本使用的是sentinel 1.4.0。为了顺利完成本文的内容，建议挑选同版本的sentinel dashboard来使用是最稳妥的。

下载地址：https://github.com/alibaba/Sentinel/releases/download/1.4.0/sentinel-dashboard-1.4.0.jar

其他版本：https://github.com/alibaba/Sentinel/releases

同以往的Spring Cloud教程一样，这里也不推荐大家跨版本使用，不然可能会出现各种各样的问题。


通过命令启动：

```bash
java -jar sentinel-dashboard-1.4.0.jar
```

sentinel-dashboard不像Nacos的服务端那样提供了外置的配置文件，比较容易修改参数。不过不要紧，由于sentinel-dashboard是一个标准的spring boot应用，所以如果要自定义端口号等内容的话，可以通过在启动命令中增加参数来调整，比如： `-Dserver.port=8888`

默认情况下，sentinel-dashboard以8080端口启动，所以可以通过访问： localhost:8080来验证是否已经启动成功

默认用户名和密码是：sentinel/sentinel

![](http://ww1.sinaimg.cn/large/006BiJ7dly1g2rei8pf5vj30q60e4q3b.jpg)


### 整合Sentinel

第一步：在Spring Cloud应用的 pom.xml中引入Spring Cloud Alibaba的Sentinel模块：

```xml
<dependencies> 
	<dependency> 
		<groupId> org.springframework.boot </groupId> 
		<artifactId> spring-boot-starter-web </artifactId> 
	</dependency> 
	<dependency> 
		<groupId> org.springframework.cloud </groupId> 
		<artifactId> spring-cloud-starter-alibaba-sentinel </artifactId> 
	</dependency> 
	<dependency> 
		<groupId> org.projectlombok </groupId> 
		<artifactId> lombok </artifactId> 
		<version> 1.18.2 </version> 
		<optional> true </optional> </dependency> 
	<dependency> 
		<groupId> org.springframework.boot </groupId> 
		<artifactId> spring-boot-starter-test </artifactId> 
		<scope> test </scope> 
	</dependency> 
</dependencies> 
```

第二步：在Spring Cloud应用中通过 spring.cloud.sentinel.transport.dashboard参数配置sentinel dashboard的访问地址，比如：


```properties
spring.application.name=alibaba-sentinel-rate-limiting
server.port=8001
# sentinel dashboard 
spring.cloud.sentinel.transport.dashboard=localhost:8080
spring.cloud.sentinel.eager=true
```


第三步：创建应用主类，并提供一个rest接口，比如：


```java
@RestController
@Slf4j
public class UserController {

	@GetMapping("/")
	public Object name() {
		log.info("helloworld");
		return "helloworld";


	}

	

}
```


第四步：启动应用，然后通过postman或者curl访问几下 localhost:8001/接口


```bash
$ curl localhost:8001
helloworld%            
```

此时，在上一节启动的Sentinel Dashboard中就可以当前我们启动的 alibaba-sentinel-rate-limiting这个服务以及接口调用的实时监控了。具体如下图所示：


![](http://ww1.sinaimg.cn/large/006BiJ7dly1g2rf7koemxj31h10pl0vl.jpg)



### 配置自定义限流规则(@SentinelResource埋点)


自定义限流规则就不是添加方法的访问路径。 配置的是@SentinelResource注解中value的值。比如@SentinelResource("resource")就是配置路径为resource


@SentinelResource 注解包含以下属性：

- value: 资源名称，必需项（不能为空）
- entryType: 入口类型，可选项（默认为 EntryType.OUT）
- blockHandler:blockHandlerClass中对应的异常处理方法名。参数类型和返回值必须和原方法一致
- blockHandlerClass：自定义限流逻辑处理类

```java

	@SentinelResource(value = "helloworld", blockHandler = "handleException", blockHandlerClass = {ExceptionUtil.class})
	@GetMapping("/")
	public Object name() {
		log.info("helloworld");
		return "helloworld";


	}

```

```java

import java.util.HashMap;
import java.util.Map;

import com.alibaba.csp.sentinel.slots.block.BlockException;

public class ExceptionUtil {

    public static Map<String,Object> handleException(BlockException ex) {
        Map<String,Object> map=new HashMap<>();
        System.out.println("Oops: " + ex.getClass().getCanonicalName());
        map.put("Oops",ex.getClass().getCanonicalName());
        map.put("msg","通过@SentinelResource注解配置限流埋点并自定义处理限流后的逻辑");
        return  map;
    }
}
```

![](/assets/20190506112533.png)




```bash
$ curl localhost:8001 -s|jq

{
  "msg": "通过@SentinelResource注解配置限流埋点并自定义处理限流后的逻辑",
  "Oops": "com.alibaba.csp.sentinel.slots.block.flow.FlowException"
}


```

基本的限流处理就完成了。 但是每次服务重启后 之前配置的限流规则就会被清空因为是内存态的规则对象.所以下面就要用到Sentinel一个特性ReadableDataSource 获取文件、数据库或者配置中心是限流规则

## 读取文件的实现限流规则

一条限流规则主要由下面几个因素组成：

* resource：资源名，即限流规则的作用对象
* count: 限流阈值
* grade: 限流阈值类型（QPS 或并发线程数）
* limitApp: 流控针对的调用来源，若为 default 则不区分调用来源
* strategy: 调用关系限流策略
* controlBehavior: 流量控制效果（直接拒绝、Warm Up、匀速排队）

SpringCloud alibaba集成Sentinel后只需要在配置文件中进行相关配置，即可在 Spring 容器中自动注册 DataSource，这点很方便。配置文件添加如下配置


```properties
spring.application.name=alibaba-sentinel-rate-limiting
server.port=8001
# sentinel dashboard 
spring.cloud.sentinel.transport.dashboard=localhost:8080
spring.cloud.sentinel.eager=true
spring.cloud.sentinel.datasource.ds1.file.file=classpath: flowrule.json
spring.cloud.sentinel.datasource.ds1.file.ruleType=flow
spring.cloud.sentinel.datasource.ds1.file.data-type=json
```

```json
[
  {
    "resource": "helloworld",
    "controlBehavior": 0,
    "count": 2,
    "grade": 1,
    "limitApp": "default",
    "strategy": 0
  }
]
```



