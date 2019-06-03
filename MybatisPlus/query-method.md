## 基本查询

```properties

# mysql
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/mp?useSSL=false&serverTimezone=GMT%2B8
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.data=classpath:db/data-mysql.sql
spring.datasource.schema=classpath:db/schema-mysql.sql
spring.datasource.continue-on-error=true
spring.datasource.initialization-mode=ALWAYS
# log
logging.level.root=warn
logging.level.com.example.demo.dao=trace
logging.pattern.console=%p%m%n
```


```java
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import com.example.demo.po.User;

import lombok.extern.slf4j.Slf4j;

@RunWith(SpringRunner.class)
@SpringBootTest
@Slf4j
public class UserMapperTest {
	
	@Autowired
	private UserMapper userMapper;

	@Test
	public void test() {
		
		User user = userMapper.selectById(1L);
		log.info("====>>>> {}", user);
		
	}
}

```
通过字段精确查询

```java
Map<String,Object> map = new HashMap<>();
map.put("name", "Tom");// key是DB的columnname,不是po的fieldname
List<User> list = userMapper.selectByMap(map);
list.forEach(u -> log.info(u.toString()));

```
通过构造器方式查询

```java
//2种构造方式
//QueryWrapper<User> queryWrapper = new QueryWrapper<>();
QueryWrapper<User> queryWrapper = Wrappers.query();



//name like %Tom% and age < 30
queryWrapper.like("name", "Tom").lt("age", 30);

//name like %Tom% and age between 20 and 40 and email is not null
queryWrapper.like("name", "Tom").between("age", 20, 40).isNotNull("email");


//name like %Tom% or age >= 40 order by age desc,id asc
queryWrapper.like("name", "Tom").or().ge("age", 25).orderByDesc("age").orderByAsc("id");

//date_format(create_time,'%Y-%m-%d') = '2019-02-14' and manager_id in (select id from user where name like 'Tom%')
queryWrapper.apply("date_format(create_time,'%Y-%m-%d') = {0}", "2019-02-14")
			.inSql("manager_id", "select id from user where name like 'Tom%'");

//name like %Tom% and (age < 25 or email is not null)
queryWrapper.like("name", "Tom").and(qw->qw.lt("age", 40).or().isNotNull("email"));

//name like %Tom% or (age < 40 and age > 20 and email is not null)
queryWrapper.like("name", "Tom").or(qw-> qw.lt("age", 40).gt("age", 20).isNotNull("email"));

//(agent < 40 or email is not null) and name like '%Tom%'
queryWrapper.nested(wq->wq.lt("age", 40).or().isNotNull("email")).like("name", "Tom");

//age in (23, 34, 45) limit 1
queryWrapper.in("age", Arrays.asList(23, 34, 45)).last("limit 1");



//不列出全出字段
//select name, age from user where ...
queryWrapper.select("name", "age");

//condition的作用，条件满足存在;不满足不存在
queryWrapper.select("name", "age").like(!StringUtils.isEmpty(user.getName()), "name", user.getName());

//实体对象出入构造器作为查询条件

User user = new User();
user.setName("Tom");
user.setAge(12);
//2种构造方式
//QueryWrapper<User> queryWrapper = new QueryWrapper<>();
QueryWrapper<User> queryWrapper = Wrappers.query(user);
List<User> list = userMapper.selectList(queryWrapper);
list.forEach(System.out::println);


//只查询指定的几列，用map接收
	
QueryWrapper<User> queryWrapper = Wrappers.query();
queryWrapper.select("name", "age");
List<Map<String,Object>> list = userMapper.selectMaps(queryWrapper);
list.forEach(System.out::println);



//只查询指定的几列，用map接收
//select avg(age) avg_age, min(age) min_age, max(age) max_age
//from user
//group by manager_id
//having sum(age) < 500

QueryWrapper<User> queryWrapper = Wrappers.query();
queryWrapper.select("avg(age) avg_age, min(age) min_age, max(age) max_age")
	.groupBy("manager_id").having("sum(age) < {0}", 500);
List<Map<String,Object>> list = userMapper.selectMaps(queryWrapper);




Lisy<User> aist = userMapper.selectList(queryWrapper);




```

## lambda条件构造器


```java
//创建方式一
LambdaQueryWrapper<Object> lambda = new QueryWrapper<>().lambda();
//创建方式二
LambdaQueryWrapper<Object> lambda = new LambdaQueryWrapper<>();
//创建方式三
LambdaQueryWrapper<Object> lambda = Wrappers.lambdaQuery();

```



```java

LambdaQueryWrapper<User> lambda = Wrappers.lambdaQuery();
lambda.select(User::getName, User::getAge).like(User::getName, "Tom").lt(User::getAge, 40);//name like '%Tom%' and age < 40
List<Map<String,Object>> list = userMapper.selectMaps(lambda);
list.forEach(System.out::println);
```

```java

// name like '%tom%' and (age < 30 or email is not null)
lambda.like(User::getName, "tom").and(lqw-> lqw.lt(User::getAge, 30).or().isNotNull(User::getEmail));
```


lambda链式调用

```java


List<User> list = new LambdaQueryChainWrapper<>(userMapper).like(User::getName, "Tom").lt(User::getAge, 40).list();
list.forEach(System.out::println);

```


## 分页插件

启用分页插件

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.baomidou.mybatisplus.extension.plugins.PaginationInterceptor;

@Configuration
public class MybatisPlusContiguration {
	@Bean
	public PaginationInterceptor paginationInterceptor() {
		return new PaginationInterceptor();
	}

}

```

```java

LambdaQueryWrapper<User> lambdaQueryWrapper = new LambdaQueryWrapper<>();
Page<User> page = new Page<>(1,2);//当前页，每页显示条数
IPage<User> iPage = userMapper.selectPage(page, lambdaQueryWrapper);//返回实体T
//IPage<Map<String,Object>> iPage2 = userMapper.selectMapsPage(page, lambdaQueryWrapper);//返回map
System.out.println(iPage.getCurrent());//当前页
System.out.println(iPage.getPages());//一共有多少页
System.out.println(iPage.getSize());//每页大小
System.out.println(iPage.getTotal());//一共有多少条
System.out.println(iPage.getRecords());//分页数据
```

```java

Page<User> page = new Page<>(1, 2, true);//true查询总条数count,false不查询总条数
```























