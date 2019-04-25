# SpringBoot整合MongoDB(1)

# springboot整合mongodb

## gradle配置

```groovy
buildscript {
	ext {
		springBootVersion = '2.0.0.RC2'
	}
	repositories {
		mavenCentral()
		maven { url "https://repo.spring.io/snapshot" }
		maven { url "https://repo.spring.io/milestone" }
	}
	dependencies {
		classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
	}
}

apply plugin: 'java'
apply plugin: 'eclipse'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'

group = 'com.mongo'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = 1.8

repositories {
	mavenCentral()
	maven { url "https://repo.spring.io/snapshot" }
	maven { url "https://repo.spring.io/milestone" }
}


dependencies {
	compile('org.springframework.boot:spring-boot-starter-data-mongodb')
	testCompile('org.springframework.boot:spring-boot-starter-test')
}
```

## mongodb自动配置

```properties
#没有开启安全认证
#spring.data.mongodb.uri=mongodb://localhost:27017/springboot-db
#设置了账户和密码
spring.data.mongodb.uri=mongodb://name:pass@localhost:27017/dbname
```

## 启动类

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
	
	public static void main(String[] args) {
		
		
		SpringApplication.run(Application.class, args);
		
	}

}
```

## 测试类

```java

import java.util.List;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.test.context.junit4.SpringRunner;
import com.mongo.Application;
import com.mongo.entity.User;

@RunWith(SpringRunner.class)
@SpringBootTest(classes = { Application.class })
public class MongoDemo {

	@Autowired
	private MongoTemplate mongoTemplate;

	@Test
	public void demoOne() throws Exception {
		
		mongoTemplate.insert(new User("woms", 12, "山西省",new String[] {"hello","mongo"},  new Integer[]{12,11} )) ;

		List<User> list = mongoTemplate.find(new Query(), User.class);
		for (User user : list) {
			System.out.println(user);
		}

	}

}
```
先来感受一下
```java
//删除所有
		//customerRepository.deleteAll();

	        // 添加数据
		//customerRepository.save(new Customer("Alice", "Smith"));
		//customerRepository.save(new Customer("Bob", "Smith"));

	        // fetch all customers
//	        System.out.println("Customers found with findAll():");
//	        System.out.println("-------------------------------");
//	        for (Customer customer : customerRepository.findAll()) {
//	            System.out.println(customer);
//	        }
//	        System.out.println();

	        // fetch an individual customer
//	        System.out.println("Customer found with findByFirstName('Alice'):");
//	        System.out.println("--------------------------------");
//	        System.out.println(customerRepository.findByFirstName("Alice"));
//
//	        System.out.println("Customers found with findByLastName('Smith'):");
//	        System.out.println("--------------------------------");
//	        for (Customer customer : customerRepository.findByLastName("Smith")) {
//	            System.out.println(customer);
//	        }
		
		//分页排序查询--repository
//		PageRequest pageRequest = new PageRequest(0,20,new Sort(Direction.ASC, "lastName"));
//		Page<Customer> page = customerRepository.findAll(pageRequest);
//		List<Customer> content = page.getContent();
//		for (Customer customer : content) {
//			System.out.println(customer);
//		}
		
		//customerRepository.findAll(example, pageRequest);
		
		//分页排序--mongoTemplate
		BasicDBObject basicDBObject = new BasicDBObject();
		//模糊查询
		basicDBObject.put("lastName", new BasicDBObject("$regex", Pattern.compile("mit")).append("$options", "i"));
		Query query = 	new BasicQuery(basicDBObject);
		query.with(new PageRequest(0, 20));
		query.with(new Sort(Direction.ASC, "lastName"));
		List<Customer> list = mongoTemplate.find(query, Customer.class);
		for (Customer customer : list) {
			System.out.println(customer);
		}
		//保存数据--mongoTemplate
		//	mongoTemplate.remove(new Query(), Customer.class);
		//删除数据
		//	mongoTemplate.insert(new Customer("Alice", "Smith"));
	
		//修改数据--mongoTemplate
		mongoTemplate.updateMulti(query, new Update().set("lastName", "wwwmmmsss"), Customer.class);
```


## 总结

有多种方式：

1. 使用原生的驱动api，也就是使用内嵌mongodb

2. 使用springboot-data-jpa

    2-1. 使用extends MongoRepository
    
    2-2. 使用mongoTemplate
    
    2-3. 使用@query注解


