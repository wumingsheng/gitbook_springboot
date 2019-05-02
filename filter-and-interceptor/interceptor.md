# 拦截器

## 1. 定义拦截器

```java
package com.example.demo.interceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import lombok.extern.slf4j.Slf4j;


@Slf4j
public class FirstInterceptor implements HandlerInterceptor {

	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		log.info("first intercrptor preHandle ...");
		return HandlerInterceptor.super.preHandle(request, response, handler);
	}

	@Override
	public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
			ModelAndView modelAndView) throws Exception {
		log.info("first intercrptor postHandle ...");
		HandlerInterceptor.super.postHandle(request, response, handler, modelAndView);
	}

	@Override
	public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
			throws Exception {
		log.info("first intercrptor afterCompletion ...");
		HandlerInterceptor.super.afterCompletion(request, response, handler, ex);
	}
	
	

}

```

## 2. 注册拦截器

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import com.example.demo.interceptor.FirstInterceptor;

@Configuration
public class InterceptorConfiguration implements WebMvcConfigurer {

	@Bean
	public HandlerInterceptor firstInterceptor() {
		return new FirstInterceptor();
	}

	@Override
	public void addInterceptors(InterceptorRegistry registry) {
		// 由于拦截器加载是在springContext创建之前完成的，如果直接new AuthInterceptor，那么在拦截器中注入实体就为会null
		registry.addInterceptor(firstInterceptor()).addPathPatterns("/**");
	}
}
```

## 3. 测试

```log
2019-05-02 11:17:35.300  INFO 23821 --- [nio-8080-exec-1] c.e.demo.interceptor.FirstInterceptor    : first intercrptor preHandle ...
2019-05-02 11:17:35.313  INFO 23821 --- [nio-8080-exec-1] c.e.demo.interceptor.FirstInterceptor    : first intercrptor postHandle ...
2019-05-02 11:17:35.313  INFO 23821 --- [nio-8080-exec-1] c.e.demo.interceptor.FirstInterceptor    : first intercrptor afterCompletion ...
```


## 4. 多个拦截器顺序

* 处理器前方法采用先注册先执行
* 处理器后方法和完成方法采用先注册后执行的顺序


![](/assets/20190502114259.png)

```java
package com.example.demo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import com.example.demo.interceptor.FirstInterceptor;
import com.example.demo.interceptor.PrometheusInterceptor;

@Configuration
public class InterceptorConfiguration implements WebMvcConfigurer {

	@Bean
	public HandlerInterceptor firstInterceptor() {
		return new FirstInterceptor();
	}
	@Bean
	public HandlerInterceptor prometheusInterceptor() {
		return new PrometheusInterceptor();
	}

	@Override
	public void addInterceptors(InterceptorRegistry registry) {
		registry.addInterceptor(firstInterceptor()).addPathPatterns("/**");
		registry.addInterceptor(prometheusInterceptor()).addPathPatterns("/**");
	}
}

```

```log
2019-05-02 11:22:48.165  INFO 24828 --- [nio-8080-exec-1] c.e.demo.interceptor.FirstInterceptor    : first intercrptor preHandle ...
2019-05-02 11:22:48.165  INFO 24828 --- [nio-8080-exec-1] c.e.d.interceptor.PrometheusInterceptor  : Prometheus intercrptor preHandle ...
2019-05-02 11:22:48.179  INFO 24828 --- [nio-8080-exec-1] c.e.d.interceptor.PrometheusInterceptor  : Prometheus intercrptor postHandle ...
2019-05-02 11:22:48.179  INFO 24828 --- [nio-8080-exec-1] c.e.demo.interceptor.FirstInterceptor    : first intercrptor postHandle ...
2019-05-02 11:22:48.179  INFO 24828 --- [nio-8080-exec-1] c.e.d.interceptor.PrometheusInterceptor  : Prometheus intercrptor afterCompletion ...
2019-05-02 11:22:48.179  INFO 24828 --- [nio-8080-exec-1] c.e.demo.interceptor.FirstInterceptor    : first intercrptor afterCompletion ...
```

