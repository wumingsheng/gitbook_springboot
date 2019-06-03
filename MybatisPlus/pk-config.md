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



















