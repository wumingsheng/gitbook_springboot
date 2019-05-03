# springboot快速整合mybatis



## 1. build.gradle

```groovy
buildscript {
	ext {
		springBootVersion = '2.0.3.RELEASE'
	}
	repositories {
		mavenCentral()
	}
	dependencies {
		classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
	}
}

apply plugin: 'java'
apply plugin: 'eclipse'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = 1.8

repositories {
	mavenCentral()
}


dependencies {
	runtime('mysql:mysql-connector-java')
	testCompile('org.springframework.boot:spring-boot-starter-test')
	compile('org.mybatis.spring.boot:mybatis-spring-boot-starter:1.3.2')
	//compile group: 'com.alibaba', name: 'druid', version: '1.1.10'
}
```
## 2. application.yml

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver 
    url: jdbc:mysql://172.18.113.222:3306/paimai?characterEncoding=UTF-8
    username: Dev1
    password: 123456
#    type: com.alibaba.druid.pool.DruidDataSource
```
## 3. application.java
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Boot2Application {

	public static void main(String[] args) {
		SpringApplication.run(Boot2Application.class, args);
	}
}
```
## 4. dao
```java
import org.apache.ibatis.annotations.Insert;
import org.apache.ibatis.annotations.Mapper;

import com.example.demo.dao.po.OrderInfo;

@Mapper
public interface OrderInfoDao {
	
	@Insert("insert into order_info (order_id, add_datetime, order_money, order_user_id, product_id, shop_id) "
			+ "VALUES (#{orderId},#{addDatetime},#{orderMoney},#{orderUserId},#{productId},#{shopId})")
	public int saveOrderInfo(OrderInfo orderInfo);

}

```
## 5. po
```java
import java.util.Date;

public class OrderInfo {
	
private String orderId;
	
	private Date addDatetime;
	
	private Double orderMoney;
	
	private String orderUserId; 
	
	private String productId;
	
	private String shopId;

	//setter,getter...
}

```
## 6. test


```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class Boot2ApplicationTests {
	
	@Autowired
	private OrderInfoDao orderInfoDao;

	@Test
	@Transactional
	public void contextLoads() throws Exception {
		
		OrderInfo orderInfo = new OrderInfo();
		orderInfo.setAddDatetime(new Date());
		orderInfo.setOrderId(UUID.randomUUID().toString());
		orderInfo.setOrderMoney(20d);
		orderInfo.setOrderUserId("007");
		orderInfo.setProductId("10010");
		orderInfo.setShopId("00254");
		int saveOrderInfo = orderInfoDao.saveOrderInfo(orderInfo);
		System.out.println(saveOrderInfo);
		Integer.parseInt("ddddd");
		OrderInfo orderInfo2 = new OrderInfo();
		orderInfo2.setAddDatetime(new Date());
		orderInfo2.setOrderId(UUID.randomUUID().toString());
		orderInfo2.setOrderMoney(20d);
		orderInfo2.setOrderUserId("007");
		orderInfo2.setProductId("10010");
		orderInfo2.setShopId("00254");
		int saveOrderInfo2 = orderInfoDao.saveOrderInfo(orderInfo2);
		System.out.println(saveOrderInfo2);
	}

}
```



## 7. 说明：

1. 可以不用配置连接池，boot2默认自带hikari连接池，不用引用连接池的jar包，也不用配置type

2. 可以不用配置`@EnableTransactionManagement`，默认启动

3. 最小配置有两种方式开启mybatis

   1. 在每一个dao上加`@mapper`注解
   2. 在启动类上加注解指定dao包`@MapperScan("com.example.demo.dao")`

4. 配置下划线和驼峰命名自动转化

   ```yaml
   mybatis:
     configuration:
       map-underscore-to-camel-case: true
       use-generated-keys: true
       use-column-label: true
     type-aliases-package: com.ly.paimai.dao.po
     mapper-locations:
     - classpath:mapper/*.xml
   ```


5. 使用注解的方式，别名配置不是很重要，但是建议还是配置上的好

6. sql语句拦截器配置，配置到yml文件中好像还不支持自定的拦截器，实例化拦截器，会自动注册的

```java
package com.ly.paimai.config;

import java.util.List;
import java.util.Properties;
import java.util.StringTokenizer;

import org.apache.commons.beanutils.BeanUtils;
import org.apache.ibatis.executor.statement.StatementHandler;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.ParameterMapping;
import org.apache.ibatis.plugin.Interceptor;
import org.apache.ibatis.plugin.Intercepts;
import org.apache.ibatis.plugin.Invocation;
import org.apache.ibatis.plugin.Plugin;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;


@Intercepts({ 
	@org.apache.ibatis.plugin.Signature(type = StatementHandler.class, method = "prepare", args = { java.sql.Connection.class,java.lang.Integer.class }) 
})
@Component
public class MybatisSqlLogInterceptor implements Interceptor {
	
	private static  final Logger log = LoggerFactory.getLogger(MybatisSqlLogInterceptor.class);
	

	

	
	

	public Object intercept(Invocation invocation) throws Exception {
		StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
		BoundSql bSql = statementHandler.getBoundSql();
		Object param = statementHandler.getParameterHandler().getParameterObject();

		StringBuffer sb = new StringBuffer();

		sb.append(new StringBuilder().append(removeBreakingWhitespace(bSql.getSql())).toString());

		List<ParameterMapping> paramList = bSql.getParameterMappings();
		for (ParameterMapping mapping : paramList) {
			String proName = mapping.getProperty();
			try {
				sb.append(new StringBuilder().append("[").append(proName).append(":")
				        .append(BeanUtils.getProperty(param, proName)).append("]").toString());
			} catch (Exception e) {
				sb.append(new StringBuilder().append("[").append(proName).append(":").append(param).append("]").toString());
			}
		}
		Object result = null;
		try {
			result = invocation.proceed();
			log.info(sb.toString());
		} catch (Exception e) {
			log.error(sb.toString(), e);
			throw e;
		}

		return result;
	}


	protected String removeBreakingWhitespace(String original) {
		StringTokenizer whitespaceStripper = new StringTokenizer(original);
		StringBuilder builder = new StringBuilder();
		for (; whitespaceStripper.hasMoreTokens(); builder.append(" ")) {
			builder.append(whitespaceStripper.nextToken());
		}
		return builder.toString();
	}


	public Object plugin(Object target) {
		return Plugin.wrap(target, this);
	}


	public void setProperties(Properties arg0) {
	}
}

```

> http://www.mybatis.org/mybatis-3/configuration.html#plugins





