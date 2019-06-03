## 1. 特性1 - 自动生成id


先定义一个普通的po

```java

@Data
public class User {
	
    private Long id;
    
    private String name;
    
    private Integer age;
    
    private String email;
}

```

我们在保存对象的时候，并没有设置id主键

```java
User user = new User();
		
user.setAge(23);
user.setEmail("2332fw@qq.com");
user.setName("woms");
userMapper.insert(user);
```

mybatis-plus会自动帮我们生成id

```log
DEBUG==>  Preparing: INSERT INTO user ( id, name, age, email ) VALUES ( ?, ?, ?, ? ) 
DEBUG==> Parameters: 1135178229880168450(Long), woms(String), 23(Integer), 2332fw@qq.com(String)
DEBUG<==    Updates: 1
```
## 特性2 - 下划线和驼峰自动转换

## 特性3 - 常用注解


主键、字段和表名的显式声明

```java

@Data
@TableName("user")
public class User {
	
	@TableId("id")
    private Long id;
    
    private String name;
    
    
    private Integer age;
    
    @TableField("email")
    private String email;
}


```

## 特性4 - 排除非表字段的三种方式

### 4.1 关键字 `transient` 标示的字段不参与序列化过程

```java
private transient String remark;
```

### 4.2 关键字 `static` 标示的字段

由于static字段属于类不属于对象，全局就一个，不符合我们的正常使用

### 4.3 `@TableField(exist = false)`

```java

@TableField(exist = false)
private String remark;

```






























