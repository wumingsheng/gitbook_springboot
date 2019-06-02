## 1. 依赖

```groovy

plugins {
    id 'org.springframework.boot' version '2.1.5.RELEASE'
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
    maven { url 'http://maven.aliyun.com/nexus/content/groups/public/'  }
    mavenCentral()
        
}

dependencies {


    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compile 'com.baomidou:mybatis-plus-boot-starter:3.1.1'
    compileOnly 'org.projectlombok:lombok'
    runtimeOnly 'mysql:mysql-connector-java'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation
    'org.springframework.boot:spring-boot-starter-test'



}

```

## 2. 配置文件

```properties
spring.datasource.url=jdbc:mysql://192.168.1.112:3306/mp?useSSL=false&serverTimezone=GMT%2B8
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.data=classpath:db/data-mysql.sql
spring.datasource.schema=classpath:db/schema-mysql.sql
spring.datasource.continue-on-error=true
spring.datasource.initialization-mode=ALWAYS
```

## 3. 启动类配置

```java
package com.example.demo;


import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

import lombok.extern.slf4j.Slf4j;

@SpringBootApplication
@Slf4j
@MapperScan("com.example.demo.dao")
public class DemoApplication {

public static void main(String[] args) {
        
    log.info("app is running...");
    SpringApplication.run(DemoApplication.class, args);
                            
}


}
```


## 4. dao接口


```java
package com.example.demo.dao;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.example.demo.po.User;

public interface UserMapper extends BaseMapper<User>{


}
```

## 5. sql文件

```sql
DROP TABLE IF EXISTS user;

CREATE TABLE user
(
    id BIGINT(20) NOT NULL COMMENT '主键ID',
    name VARCHAR(30) NULL DEFAULT NULL COMMENT '姓名',
    age INT(11) NULL DEFAULT NULL COMMENT '年龄',
    email VARCHAR(50) NULL DEFAULT NULL COMMENT '邮箱',
    PRIMARY KEY (id)
                    
);
```

```sql
DELETE FROM user;

INSERT INTO user (id, name, age, email) VALUES
(1, 'Jone', 18, 'test1@baomidou.com'),
(2, 'Jack', 20, 'test2@baomidou.com'),
(3, 'Tom', 28, 'test3@baomidou.com'),
(4, 'Sandy', 21, 'test4@baomidou.com'),
(5, 'Billie', 24, 'test5@baomidou.com');
```
















