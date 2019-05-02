# 过滤器

两种方式： 

1. 使用spring boot提供的FilterRegistrationBean注册Filter 
2. 使用原生servlet注解`@WebFilter`、`@Order`定义Filter 

两种方式的本质都是一样的，都是去FilterRegistrationBean注册自定义Filter

## 方式一

### 1. 定义filter

```java
import java.io.IOException;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;

import lombok.extern.slf4j.Slf4j;
@Slf4j
public class FirstFilter implements Filter {

	@Override
	public void init(FilterConfig filterConfig) throws ServletException {
		// TODO Auto-generated method stub
		Filter.super.init(filterConfig);
	}

	@Override
	public void destroy() {
		// TODO Auto-generated method stub
		Filter.super.destroy();
	}

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		log.info("first filter doFilter===");
		chain.doFilter(request, response);
		
	}

}

```

```java
package com.example.demo.filter;

import java.io.IOException;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;

import lombok.extern.slf4j.Slf4j;
@Slf4j
public class SecondFilter implements Filter {

	@Override
	public void init(FilterConfig filterConfig) throws ServletException {
		// TODO Auto-generated method stub
		Filter.super.init(filterConfig);
	}

	@Override
	public void destroy() {
		// TODO Auto-generated method stub
		Filter.super.destroy();
	}

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		log.info("second filter doFilter===");
		chain.doFilter(request, response);
		
	}

}

```

### 2. 注册filter

```java
package com.example.demo.config;

import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.example.demo.filter.FirstFilter;
import com.example.demo.filter.SecondFilter;

@Configuration
public class FilterConfig {
	


	
	@Bean
    public FilterRegistrationBean<FirstFilter> FirstFilter() {
        FilterRegistrationBean<FirstFilter> filterRegistrationBean = new FilterRegistrationBean<>();
        filterRegistrationBean.setFilter(new FirstFilter());
        filterRegistrationBean.addUrlPatterns("/*");
        filterRegistrationBean.setName("FirstFilter");
        filterRegistrationBean.setOrder(1);
        return filterRegistrationBean;
    }
	@Bean
	public FilterRegistrationBean<SecondFilter> SecondFilter() {
		FilterRegistrationBean<SecondFilter> filterRegistrationBean = new FilterRegistrationBean<>();
		filterRegistrationBean.setFilter(new SecondFilter());
		filterRegistrationBean.addUrlPatterns("/*");
		filterRegistrationBean.setName("SecondFilter");
		filterRegistrationBean.setOrder(2);
		return filterRegistrationBean;
	}


}


```

## 方式二


```java
import java.io.IOException;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.annotation.WebFilter;

import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import lombok.extern.slf4j.Slf4j;
@Slf4j
@Component
@WebFilter(filterName = "ThirdFilter" ,urlPatterns = "/*")
@Order(3)
public class ThirdFilter implements Filter {


	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		log.info("third filter doFilter===");
		chain.doFilter(request, response);
		
	}

}
```

当然也可以混合使用

```log
2019-05-02 12:19:32.770  INFO 5996 --- [nio-8080-exec-1] com.example.demo.filter.FirstFilter      : first filter doFilter===
2019-05-02 12:19:32.770  INFO 5996 --- [nio-8080-exec-1] com.example.demo.filter.SecondFilter     : second filter doFilter===
2019-05-02 12:19:32.770  INFO 5996 --- [nio-8080-exec-1] com.example.demo.filter.ThirdFilter      : third filter doFilter===
```











