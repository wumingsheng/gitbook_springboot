# 快速开始

## 依赖

不需要额外的依赖包，集成在`spring-context`内

## 配置

```java
import java.util.concurrent.Executor;

import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.AsyncConfigurer;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;


@Configuration
@EnableAsync//开启异步线程功能
public class AsyncConfig implements AsyncConfigurer {

	@Override
	public Executor getAsyncExecutor() {
		//定义线程池
		ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
		//核心线程数
		taskExecutor.setCorePoolSize(10);
		//线程池最大线程数
		taskExecutor.setMaxPoolSize(30);
		//线程队列最大线程数
		taskExecutor.setQueueCapacity(2000);
		//初始化
		taskExecutor.initialize();
		return taskExecutor;
	}
	
	

}
```

## 使用

开启一部线程使用`@Async`


```java
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import lombok.extern.slf4j.Slf4j;

@Service
@Slf4j
public class SayHelloService {
	
	@Async
	public void sayHello() {
		
		log.info("=============start async task....");
		//your code here
		
	}

}

```

## 结果

可以看出来，执行`@Async`方法时，使用的是我们自己线程池中的线程执行的

```log
2019-05-03 22:11:52.917  INFO 8514 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2019-05-03 22:11:52.917  INFO 8514 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2019-05-03 22:11:52.923  INFO 8514 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 6 ms
2019-05-03 22:11:52.938  INFO 8514 --- [nio-8080-exec-1] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService
2019-05-03 22:11:52.942  INFO 8514 --- [lTaskExecutor-1] c.e.asyncdemo.service.SayHelloService    : =============start async task....
```



