# redis-cache

## 1. 依赖

```groovy
implementation 'org.springframework.boot:spring-boot-starter-actuator'
implementation 'org.springframework.boot:spring-boot-starter-web'
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:2.0.1'
compileOnly 'org.projectlombok:lombok'
runtimeOnly 'mysql:mysql-connector-java'
annotationProcessor 'org.projectlombok:lombok'

implementation 'org.apache.commons:commons-lang3:3.9'
implementation 'commons-beanutils:commons-beanutils:1.9.3'

compile group: 'tk.mybatis', name: 'mapper-spring-boot-starter', version: '2.1.5'
```

## 2. 配置

```properties
#---mysql---
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/demo?characterEncoding=UTF-8
spring.datasource.username=root
spring.datasource.password=123456
#---mybatis---
mybatis.mapper-locations=classpath:mapper/*.xml
mybatis.type-aliases-package=com.example.mybatisdemo.po
mybatis.type-handlers-package=com.example.mybatisdemo.typehandler
mybatis.configuration.map-underscore-to-camel-case=true
mybatis.configuration.interceptors=
#---mapper---
mapper.mappers=com.example.mybatisdemo.dao.BaseMapper
mapper.not-empty=false
mapper.identity=MYSQL
mapper.enum-as-simple-type=true
#---cache---
spring.cache.cache-names=redisCache
spring.cache.redis.key-prefix=demo:
spring.cache.redis.cache-null-values=true
spring.cache.redis.time-to-live=0ms
spring.cache.redis.use-key-prefix=true
spring.cache.type=REDIS
#---redis---
spring.redis.host=127.0.0.1
spring.redis.port=6379
spring.redis.database=0
spring.redis.jedis.pool.max-active=10
spring.redis.jedis.pool.max-idle=10
spring.redis.jedis.pool.min-idle=5
spring.redis.jedis.pool.max-wait=2000
```

## 3. 启动类

1. 因为我们使用了`tk.mybatis`作通用的mapper,这里的`MapperScan`要使用tk的
2. 开启缓存使用`@EnableCaching`

```java
//import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.stereotype.Repository;

import tk.mybatis.spring.annotation.MapperScan;

@SpringBootApplication
@MapperScan(value = "com.example.mybatisdemo.dao", annotationClass = Repository.class)
@EnableCaching
public class MybatisDemoApplication {
	

	public static void main(String[] args) {
		SpringApplication.run(MybatisDemoApplication.class, args);
	}

}
```

## 4. mybatis日志插件

```java
package com.example.mybatisdemo.component;


import java.sql.Connection;
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
import org.apache.ibatis.plugin.Signature;
import org.springframework.stereotype.Component;

import lombok.extern.slf4j.Slf4j;

@Slf4j
@Component
@Intercepts({@Signature(type = StatementHandler.class, method = "prepare",args = { Connection.class,Integer.class }) })
public class MybatisSqlLogInterceptor implements Interceptor {

   

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

## 5. rest

```java

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.context.annotation.Scope;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import com.example.mybatisdemo.po.User;
import com.example.mybatisdemo.service.UserService;

import lombok.extern.slf4j.Slf4j;

@RestController
@RequestMapping("user")
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
@Slf4j
public class UserController {
	
	
	@Autowired
	private UserService userService;
	
	//curl localhost:8080/user/get/2
	@GetMapping("get/{id}")
	public Object getUser(@PathVariable(value = "id", required = true) String id) throws Exception {
		log.info("=======id={}", id);
		 User user = userService.getUser(id);
		 log.info("======user={}", user);
		 return user;
	}
	//curl localhost:8080/user/save -X POST -H "Content-Type: application/json" 
	//--data '{"id":"2","username":"jack","password":"123","status":"ENABLED"}'
	@PostMapping("save")
	public Object getUser(@RequestBody User user) throws Exception {
	
		 log.info("======user={}", user);
		 return userService.saveUser(user);
	}
	@PostMapping("delete/{id}")
	public Object deleteUser(@PathVariable(value = "id", required = true) String id) throws Exception {
		
		log.info("=======id={}", id);
		 int deleteUser = userService.deleteUser(id);
		 return deleteUser;
	}

}

```

## 6. dao

```java
import tk.mybatis.mapper.common.ConditionMapper;
import tk.mybatis.mapper.common.IdsMapper;
import tk.mybatis.mapper.common.Mapper;
import tk.mybatis.mapper.common.MySqlMapper;

public interface BaseMapper<T> extends Mapper<T>, MySqlMapper<T>, IdsMapper<T>, ConditionMapper<T> {
}
```

```java
import org.springframework.stereotype.Repository;

import com.example.mybatisdemo.po.User;


@Repository
public interface UserDao extends BaseMapper<User> {
	
	User getUserById(String id);

}

```

## 7. enum

po里面定义了枚举，json返回的是这个枚举的拼写

```java
import lombok.AllArgsConstructor;
import lombok.Getter;

@AllArgsConstructor
@Getter
public enum StatusEnum {

	ENABLED(1, "enabled"), DISABLED(2, "disabled");

	private Integer statusCode;
	

	private String statusDesc;
	

	public static String getDesc(Integer statusCode) {

		for (StatusEnum en : StatusEnum.values()) {
			if (en.statusCode.equals(statusCode)) {
				return en.statusDesc;
			}
		}
		return null;

	}
	

	
	public static StatusEnum getEnumByCode(Integer code) {
		for (StatusEnum en : StatusEnum.values()) {
			if (en.statusCode.equals(code)) {
				return en;
			}
		}
		
		return null;
	}

}

```

## 8. po

缓存的对象必须实现序列化

```java
import java.io.Serializable;

import javax.persistence.Id;

import org.apache.ibatis.type.Alias;

import com.example.mybatisdemo.enums.StatusEnum;

import lombok.Data;
//要想缓冲 这里必须实现序列化  
@Data
@Alias("user")
public class User implements Serializable {
	
	/**
	 * 
	 */
	private static final long serialVersionUID = -7411979954615843818L;

	@Id
	private String id;
	
	private String username;
	
	private String password;
	
	private StatusEnum status;

}

```

## 9. service

缓存注解中的value就是配置文件中定义的`spring.cache.cache-names=redisCache`

```java
package com.example.mybatisdemo.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.example.mybatisdemo.dao.UserDao;
import com.example.mybatisdemo.po.User;

@Service
@Transactional
public class UserService {
	
	@Autowired
	private UserDao userDao;
	
	@Cacheable(value = "redisCache", key = "'redis_user_'+#id")
	public User getUser(String id) {
		return userDao.getUserById(id);
	}
	@CachePut(value = "redisCache", key = "'redis_user_'+#result.id")
	public User saveUser(User user) {
		 userDao.insert(user);
		 return user;
	}
	@CacheEvict(value = "redisCache", key = "'redis_user_'+#id", beforeInvocation = false)
	public int deleteUser(String id) {
		return userDao.deleteByPrimaryKey(id);
	}
	@CachePut(value = "redisCache", key = "'redis_user_'+#result.id", condition = "#result != 'null'")
	public User updateUser(User user) {
		User user2 = getUser(user.getId());
		
		if(null == user2) {
			return null;
		}
		
		 userDao.updateByPrimaryKeySelective(user);
		 return user;
	}

}

```


## 10. typeHandler

```java
import java.sql.CallableStatement;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

import org.apache.ibatis.type.BaseTypeHandler;
import org.apache.ibatis.type.JdbcType;
import org.apache.ibatis.type.MappedJdbcTypes;
import org.apache.ibatis.type.MappedTypes;

import com.example.mybatisdemo.enums.StatusEnum;

@MappedJdbcTypes(JdbcType.INTEGER)
@MappedTypes(StatusEnum.class)
public class StatusTypeHandler extends BaseTypeHandler<StatusEnum> {

	@Override
	public void setNonNullParameter(PreparedStatement ps, int idx, StatusEnum statusEnum, JdbcType jdbcType)
			throws SQLException {
		ps.setInt(idx, statusEnum.getStatusCode());
		
	}

	@Override
	public StatusEnum getNullableResult(ResultSet rs, String columnName) throws SQLException {
		int statusCode = rs.getInt(columnName);
		return StatusEnum.getEnumByCode(statusCode);
	}

	@Override
	public StatusEnum getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
		int statusCode = rs.getInt(columnIndex);
		return StatusEnum.getEnumByCode(statusCode);
	}

	@Override
	public StatusEnum getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
		int statusCode = cs.getInt(columnIndex);
		return StatusEnum.getEnumByCode(statusCode);
	}

}

```









