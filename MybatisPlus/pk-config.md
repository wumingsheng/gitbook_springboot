## AR模式

ActiveRecord模式

1. po继承`Model`类
2. mapper继承`BaseMapper`


```java


import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import com.baomidou.mybatisplus.extension.activerecord.Model;

import lombok.Data;
import lombok.EqualsAndHashCode;

@Data
@EqualsAndHashCode(callSuper = false)
@TableName("user")
public class User extends Model<User> {
	
	/**
	 * 
	 */
	private static final long serialVersionUID = 7664024626111370961L;

	@TableId("id")
    private Long id;
    
    private String name;
    
    
    private Integer age;
    
    @TableField("email")
    private String email;
    
    
    @TableField(exist = false)
    private String remark;
    
}


```



```java


import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.example.demo.po.User;

public interface UserMapper extends BaseMapper<User>{

}


```
插入数据

```java

		User user = new User();
		user.setAge(12);
		user.setEmail("wu_mingsheng@126.com");
		user.setName("woms");
		boolean insert = user.insert();
		System.out.println(insert);
```

查询

```java

User user = new User().selectById(1L);
		System.out.println(user);


//方式二

	User user = new User();
		user.setId(1L);
		User user2 = user.selectById();
```


更新

```java

	User user = new User();
		user.setId(1L);
		user.setName("woms");
		user.updateById();
```







## 主键策略

局部主键策略

```java

    @TableId(value = "id", type = IdType.ID_WORKER)
    private Long id;

```
主键会自动帮我们set回po里面


全局主键配置

```properties
## mybatis-plus
mybatis-plus.global-config.db-config.id-type=uuid
# mybatis-plus.mapper-locations=```properties

```

## 通用service


```java

import com.baomidou.mybatisplus.extension.service.IService;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.example.demo.dao.UserMapper;
import com.example.demo.po.User;


@Service
public class UserService extends ServiceImpl<UserMapper, User> implements IService<User> {


}
```



```java

th(SpringRunner.class)
@SpringBootTest
public class UserMapperTest {
	
	@Autowired
	private UserService userService;

	@Test
	public void test() {
		//false 如果数据记录多余一条，不会抛出异常，取第一条
		userService.getOne(Wrappers.<User>lambdaQuery().like(User::getName, "tom"), false);
	}
}

```



