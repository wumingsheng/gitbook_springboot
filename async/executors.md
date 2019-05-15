# 线程池


```java


import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.ThreadPoolExecutor.AbortPolicy;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.AsyncConfigurerSupport;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import lombok.Generated;
import lombok.extern.slf4j.Slf4j;

/**
 * @author lichunpeng
 */
@Configuration
@Slf4j
@Generated
public class AsyncConfig extends AsyncConfigurerSupport {

    @Bean(name = "threadPoolTaskExecutor")
    public ThreadPoolTaskExecutor getAsyncThreadPoolTaskExecutor() {
        ThreadPoolTaskExecutor threadPool = new ThreadPoolTaskExecutor();
        threadPool.setCorePoolSize(10);
        threadPool.setMaxPoolSize(60);
        threadPool.setKeepAliveSeconds(60);
        threadPool.setQueueCapacity(10000);
        threadPool.setWaitForTasksToCompleteOnShutdown(true);
        threadPool.setAwaitTerminationSeconds(60);
        threadPool.initialize();
        return threadPool;
    }
    
    @Bean
    public ThreadPoolExecutor echartExecutor(@Autowired @Qualifier("echartThreadFactory") ThreadFactory threadFactory) {
    	log.info("ThreadPoolExecutor echartExecutor init ... ");
        ThreadPoolExecutor threadPoolExecutor = 
        		new ThreadPoolExecutor(9, 27, 30, TimeUnit.MINUTES, new ArrayBlockingQueue<>(10000),
        				threadFactory, new AbortPolicy());
        threadPoolExecutor.prestartAllCoreThreads(); // 预启动所有核心线程
        log.info("ThreadPoolExecutor echartExecutor init completed  !!! ");
        return threadPoolExecutor;
    }
    
    @Bean("echartThreadFactory")
    public ThreadFactory getThreadFactory() {
    	return new ThreadFactory() {
    		private final AtomicInteger mThreadNum = new AtomicInteger(1);
			@Override
			public Thread newThread(Runnable r) {
				Thread t = new Thread(r, "echart-thread-" + mThreadNum.getAndIncrement());
	            log.info(t.getName() + " has been created");
	            return t;
			}
    		
    	};
    }
    

}

```


