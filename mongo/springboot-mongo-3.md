# SpringBoot整合MongoDB(3)


## 1. 索引注解

### 1-1. @Indexed

声明该字段需要加索引，加索引后以该字段为条件检索将大大提高速度
唯一索引的话是`@Indexed(unique = true)`
过期索引的话是`@Indexed(expireAfterSeconds = 300)`相当于`db.car_position.ensureIndex({"datetime":1},{expireAfterSeconds:300})`
也可以对数组进行索引，如果被索引的列是数组时，mongodb会索引这个数组中的每一个元素。

```java
@Indexed
private String uid;
```

### 1-2. @GeoSpatialIndexed

地理位置索引

### 1-3. @TextIndexed

全文索引

### 1-4. @CompoundIndex



复合索引，加复合索引后通过复合索引字段查询将大大提高速度。

```java
@Document(collection="users")
@CompoundIndexes({
    @CompoundIndex(name = "age_idx", def = "{'name': 1, 'age': -1}")
})
public class Users  implements Serializable{
    private static final long serialVersionUID = 1L;
    ...省略代码
```

还有一个兄弟注解`@CompoundIndexes`

## 2. 其他注解

### 2-1. @Document

标注在实体类上，与hibernate异曲同工。

```java
@Document(collection="users")
public class Users  implements Serializable{
    private static final long serialVersionUID = 1L;
    //...省略代码
```

### 2-2. @Id

MongoDB默认会为每个document生成一个 _id 属性，作为默认主键，且默认值为ObjectId,可以更改 _id 的值(可为空字符串)，但每个document必须拥有 _id 属性。
当然，也可以自己设置@Id主键，不过官方建议使用MongoDB自动生成。

### 2-3. @Transient

被该注解标注的，将不会被录入到数据库中。只作为普通的javaBean属性。

```java

@Transient
private String address;
```

### 2-4. @Field

代表一个字段，可以不加，不加的话默认以参数名为列名。

```java
@Field("firstName")
private String name;
```

当然了，以上的以上，可能仅仅是冰山一角，还有很多特性等待大家去挖掘。

### 2-5. @DBRef

声明类似于关系数据库的关联关系。ps：暂不支持级联的保存功能，当你在本实例中修改了DERef对象里面的值时，单独保存本实例并不能保存DERef引用的对象，它要另外保存

### 2-6. @PersistenceConstructor

声明构造函数，作用是把从数据库取出的数据实例化为对象。该构造函数传入的值为从DBObject中取出的数据

## 3. 范例

```java
@Document(collection = "car_position")
public class CarPosition {
 @Id
 private String _id;
 @Field("datetime")
 @Indexed(name = "datetime_1",expireAfterSeconds = 300)
 private Date date;
 @Field("car_id")
 @Indexed(name = "car_id_1")
 private String carId;
 @Field("loc")
 @GeoSpatialIndexed(name = "loc_2dsphere",type = GeoSpatialIndexType.GEO_2DSPHERE)
 private Location loc;
 //setters、getters
}
```

```java
public class Location {
 @Field("type")
 private  String type;
 @Field("coordinates")
 private Double[] coordinates;
 //setters、getters
```