# SpringBoot整合MongoDB(2)


在用basedbobject拼接json对象时，当json特别长时，下面这个方法是一个合适的方法
```java
String groupStr = "{\"$group\":{\"_id\":\"$name\",\"count\":{\"$sum\":1},\"avg_age\":{\"$avg\":\"$age\"},\"total_age\":{\"$sum\":\"$age\"},\"max_age\":{\"$max\":\"$age\"},\"min_age\":{\"$min\":\"$age\"}}}";
        DBObject dbObject = (DBObject) JSON.parse(groupStr);
        
```

## 1. 插入数据



### 1-1. 插入一条数据

```java
User user = new User();
user.setName("woms");
user.setAge(15);
//db.user.insert({"name":"woms","age":15})
mongoTemplate.insert(user);
```

### 1-2. 插入多条数据

```java
//> db.user.insert([{"username":"jj","age":11},{"username":"jj","age":11}])

List<User> list = new ArrayList<>();
User user = new User();
user.setAge(14);
user.setName("jj");
list.add(user);
mongoTemplate.insertAll(list);
```

## 2. 查询数据

db.collectionName.find({查询条件}，{控制要显示的字段，可选项})
数据查询也是使用json形式设置的相等关系
可选项中要显示的字段设置为1，不想显示的字段设置为0
对于查询而言，使用java-api的find()函数，只要知道bson写法，就能写api

### 2-1. 不带查询条件--查询全部

不带查询条件，查询全部数据

```java
//db.user.find()
List<User> findAll = mongoTemplate.findAll(User.class);
// db.user.find({})
mongoTemplate.find(new Query(), User.class);
```
### 2-2. 不带查询条件--查询第一条

不带查询条件

```java
//		> db.user.findOne()
User user = mongoTemplate.findOne(new Query(), User.class);
```

### 2-3. 可选项，控制要查询的字段

```java
//> db.user.find({"name":"jj"},{"_id":0,"name":1})
//{ "name" : "jj" }

以下两种方式同效果

//方式一
mongoTemplate.find(new BasicQuery("{\"name\":\"jj\"}","{\"_id\":0,\"name\":1}"), User.class).stream().forEach(System.err::println);

//方式二
mongoTemplate.find(new BasicQuery(new BasicDBObject("name", "jj"),new BasicDBObject("_id", 0).append("name", 1)), User.class)
            .stream().forEach(System.err::println);
//User [id=null, name=jj, age=null, location=null, arrStr=null, arrInt=null]查询出结果只有name有值
```


### 2-4. 关系查询

大于（"$gt"）、小于（"$lt"）、大于等于（"$gte"）、小于等于（"$lte"）、不等于（"$ne"）、等于（{"key":"value"}、"$eq"）、模（"$mod"）

求模bson结构，`db.user.find({"age":{"$mod":[20,1]}})`，查询出所有年龄age除以20余数为1的数据记录,和其他的关系运算符不同，求模需要两个
数据运算，所以用数字表示

范例：查询name=woms95 and age >= 95的记录
```java
//相等关系可以用{key：value}也可以使用"$eq"操作符，以下两种写法等效
//> db.user.find({"age":{"$gte":95},"name":{"$eq":"woms-95"}})
//> db.user.find({"age":{"$gte":95},"name":"woms-95"})

mongoTemplate.find(new BasicQuery("{\"age\":{\"$gte\":95},\"name\":{\"$eq\":\"woms-95\"}}"), User.class).stream().forEach(System.err::println);



//{"age":{"$gte":95},"name":{"$eq":"woms-95"}}这个json串可以使用下面BasicDBObject对象
new BasicDBObject("age", new BasicDBObject("$gte", 95)).append("name", new BasicDBObject("$eq", "woms-95"))

//也可以使用下面这种构造方法

Map<String,Object> map = new HashMap<>();
map.put("age", new BasicDBObject("$gte", 95));
map.put("name", new BasicDBObject("$eq", "woms-95"));
BasicDBObject basicDBObject = new BasicDBObject(map);

```
#### 心得：

1. 一个json串“{}”就是一个BasicDBObject对象，查询条件可以使用json串，也可以使用BasicDBObject对象
2. json串中逗号相隔的key：value，使用append(key,value)拼接
3. 语法是json语法，格式是bson格式


### 2-5. 逻辑查询

与（$and）、或（$or）、非（$not 、$nor）

bson结构中，逻辑运算付的级别最高

范例：name=woms-35 or age = 99

```java

//> db.user.find({"$or":[{"name":"woms-33"},{"age":99}]})


//		方式一
mongoTemplate.find(new BasicQuery("{\"$or\":[{\"name\":\"woms-33\"},{\"age\":99}]}"), User.class).stream().forEach(System.err::println);
//		方式二
List<BasicDBObject> list = new ArrayList<>();
list.add(new BasicDBObject("name","woms-33"));
list.add(new BasicDBObject("age",99));
BasicDBObject basicDBObject = new BasicDBObject("$or", list);
mongoTemplate.find(new BasicQuery(basicDBObject), User.class).stream().forEach(System.err::println);

```

#### 心得

1. 一个{}就是一个BasicDBObject,{}的嵌套就是BasicDBObject的嵌套
2. 嵌套中如果嵌套{}，还用BasicDBObject如果嵌套[]就使用list
3. nor是一个或（or）的求反功能

加深理解
```java
//> > db.user.find({"$or":[{"name":"woms-33"},{"age":{"$gte":95}}]})
//方式一
new BasicQuery("{\"$or\":[{\"name\":\"woms-33\"},{\"age\":{\"$gte\":95}}]}")

//方式二
List<BasicDBObject> list = new ArrayList<>();
list.add(new BasicDBObject("name","woms-33"));
list.add(new BasicDBObject("age",new BasicDBObject("$gte", 95)));
new BasicQuery(new BasicDBObject("$or", list));
```

### 2-6. 范围查询

在范围内（"$in"）、不在范围内（"$nin"）

范围的内容有多个，所以要用数组表示

范例：查询age范围在[1,3,5,7,9]内的所有记录数据

```java
//	> db.user.find({"age":{"$in":[1,3,5,7,9]}})
		
mongoTemplate.find(new BasicQuery("{\"age\":{\"$in\":[1,3,5,7,9]}}"), User.class).stream().forEach(System.err::println);

//方式二
BasicDBObject basicDBObject = new BasicDBObject("age",new BasicDBObject("$in",Arrays.asList(1,3,5,7,9)));
mongoTemplate.find(new BasicQuery(basicDBObject), User.class).stream().forEach(System.err::println);
```


### 2-7.  数组查询

在MongoDB中支持数组保存，一旦支持了数组保存，就需要针对数组的数据进行匹配
可以使用MongoDB的BSON的几个操作符：all、size、slice、elemMathc

范例1：查询同时参加语文和数学的学生{“$all”,[内容1,内容2,…]}

```java
//查询数组中包含aa和dd的数据记录
//> db.user.find({"arr_str":{"$all":["aa","dd"]}})
//方式一
mongoTemplate.find(new BasicQuery("{\"arr_str\":{\"$all\":[\"aa\",\"dd\"]}}"), User.class).stream().forEach(System.err::println);
//方式二
mongoTemplate.find(new BasicQuery(new BasicDBObject("arr_str",new BasicDBObject("$all", Arrays.asList("aa","dd")))), User.class).stream().forEach(System.err::println);
```


范例2：控制数组显示的内容

```java
//控制数组返回的个数2：表示前两个，-2：表示后两个，[1,2]:表示第一个到第二个元素
//因为这个只是控制显示映射的结果集，所以放在可选项的位置，不在查询条件内
//> db.user.find({"name":"woms"},{"arr_str":{"$slice":2}})

mongoTemplate.find(new BasicQuery("{\"name\":\"woms\"}", "{\"arr_str\":{\"$slice\":2}}"), User.class).stream().forEach(System.err::println);
```

范例3：指定索引位置元素查询

```java
//> db.user.find({"arr_str.1":"bb"})
mongoTemplate.find(new BasicQuery("{\"arr_str.1\":\"bb\"}"), User.class).stream().forEach(System.err::println);
mongoTemplate.find(new BasicQuery(new BasicDBObject("arr_str.1", "bb")), User.class).stream().forEach(System.err::println);
```



### 2-9. 模糊查询-正则查询

基础语法：`{key:正则标记}`
完整语法：`{key:{“$regex":正则标记,"regex":正则标记,"$options”:"i"}}`

对于options主要是设置正则的信息查询的标记：

“i” :忽略字母大小写;
“m” :多行查找;
“x” :空白字符除了被转义的或者在字符类中以外的完全被忽略;
“s” :匹配所有的字符（圆点、”.”），包括换行内容


```
两种写法效果相同，但是第一种用java-api没找到怎么写一直包json解析异常
因为第一种写法不能加双引号，使用“/”标示正则表达式，如果加了双引号就成了字符串匹配了

> db.user.find({"name":/wo/i})
> db.user.find({"name":{"$regex":"wo","$options":"i"}})


mongoTemplate.find(new BasicQuery("{\"name\":{\"$regex\":\"wo\",\"$options\":\"i\"}}"), User.class).stream().forEach(System.err::println);


		
//	BasicDBObject basicDBObject = new BasicDBObject("name", new BasicDBObject("$regex", "wo").append("$options", "i"));
//	BasicDBObject basicDBObject = new BasicDBObject("name", new BasicDBObject("$regex", Pattern.compile("wo")).append("$options", "i"));
mongoTemplate.find(new BasicQuery(basicDBObject), User.class).stream().forEach(System.err::println);


```




### 2-10. 排序分页

#### 方式一
```java

	// db.user.find().skip(5).limit(5).sort({"age":-1})
		
		BasicQuery basicQuery = new BasicQuery("{}");
		basicQuery.setSortObject(new BasicDBObject("age", -1));
		mongoTemplate.find(basicQuery.skip(5).limit(5), User.class).stream().forEach(System.err::println);
		
		mongoTemplate.find(new Query().skip(5).limit(5).with(new Sort(Direction.ASC, "age")), User.class).stream().forEach(System.err::println);

```
#### 方式二

```java



        //分页排序--mongoTemplate
        BasicDBObject basicDBObject = new BasicDBObject();
        //模糊查询
        basicDBObject.put("lastName", new BasicDBObject("$regex", Pattern.compile("mit")).append("$options", "i"));
        Query query =     new BasicQuery(basicDBObject);
        query.with(new PageRequest(0, 20));
        query.with(new Sort(Direction.ASC, "lastName"));
        List<Customer> list = mongoTemplate.find(query, Customer.class);
        for (Customer customer : list) {
            System.out.println(customer);
        }

```

### 2-11. 距离索引查询

1. 必须创建地理位置索引，而且在插入数据之前就预先创建好`db.car_info.createIndex( { "location": "2dsphere" } )`
2. 索引字段使用的是geojson数据格式,格式如下，注意type的首字母是大写
```
location: {
type: "Point",
coordinates: [-73.856077, 40.848447]
}
```

范例１：计算点[115.21,40.31]半径为１英里范围内的所有点，这里１英里＝1.6公里　3963.2为大约地球半径

```java
//db.car_info.find({"location":{"$geoWithin":{"$centerSphere":[[115.21,40.31],1/3963.2]}}})

BasicDBObject basicDBObject = new BasicDBObject();
basicDBObject.put("location", new BasicDBObject("$geoWithin",new BasicDBObject("$centerSphere",Arrays.asList(Arrays.asList(115.21,40.31),1/3963.2))));
mongoTemplate.find(new BasicQuery(basicDBObject), CarInfo.class).stream().forEach(System.err::println);
```

注意：
`[[115.21,40.31],1/3963.2]`java-json解析异常了，需要使用BasicDBObject结合数组使用


范例２：查询距离最近的点并且排序

```java
> db.car_info.find({"location":{"$near":{"$geometry":{"type":"Point",coordinates:[115.21,40.31]},"$maxDistance":10000}}})
```


## 3. 修改数据

upsert ： 如果更新的数据不存在，就增加一条新的内容（true为增加，false不增加），默认false
multi ：表示是否只更新满足条件的第一条记录（false：只更新第一条记录，ture全部更新）默认false

### 3-1. 整行更新

整行更新，只跟新满足条件的第一条数据

```java
//这三个写法是等效的
//db.user.update({"name":"woms"},{"username":"www"})
//db.user.update({"name":"woms"},{"username":"www"},{"multi":false})
//db.user.updateOne({"name":"woms"},{"username":"www"})

//下面两个写法是等效的
mongoTemplate.updateFirst(new BasicQuery(new BasicDBObject("name","woms")), new BasicUpdate("{\"username\":\"www\"}"), User.class);
mongoTemplate.updateFirst(new BasicQuery(new BasicDBObject("name","woms")), new BasicUpdate(new BasicDBObject("username","www")), User.class);
```


多行同时更新（注意：多行更新不能整行覆盖，只能指定字段指定更新，这也正符合开发的规则）
```java
//这两个写法是等效的
//db.user.update({"name":"woms"},{"$set":{"username":"hhh"}},{"multi":true})
//db.user.updateMany({"username":"ggg"},{"$set":{"username":"hhh"}})

//下面两个写法是等效的
mongoTemplate.updateMulti(new BasicQuery("{\"username\":\"hhh\"}"), new BasicUpdate("{\"$set\":{\"username\":\"kkk\"}}"), User.class);
mongoTemplate.updateMulti(new BasicQuery(new BasicDBObject("username","www")), new Update().set("username", "ggg"), User.class);
```

跟新，不存在就插入数据
```java

//> db.user.update({"username":"vvv"},{"username":"bbb"},{"upsert":true})
mongoTemplate.upsert(new BasicQuery(new BasicDBObject("username", "bbbppp")), new BasicUpdate(new BasicDBObject("username", "oookkklll")), User.class);

```






## 4. 删除数据

### 4-1. 根据条件删除

json串内的字符串可以用双引号+转义字符“\”,也可以使用单引号

```java
//db.user.remove({"_id":ObjectId("5a9517012c2f06515dcc5d46")})

//方式一
WriteResult writeResult = mongoTemplate.remove(new BasicQuery("{'_id' : \"5a95172b81c7c82823542df2\"}"), User.class);

//方式二
User user = new User();
user.setId("5a951cf081c7c82823542df3");
WriteResult writeResult = mongoTemplate.remove(user);

//方式三
BasicDBObject basicDBObject = new BasicDBObject("_id", "5a951de881c7c82823542df4");
BasicQuery basicQuery = new BasicQuery(basicDBObject);
WriteResult writeResult = mongoTemplate.remove(basicQuery, User.class);
```

### 4-2. 删除全部数据

```java
//> db.user.remove({})
WriteResult writeResult = mongoTemplate.remove(new Query(), User.class);
```    

## 5. 聚合操作

### 5-1. 统计个数

count({})函数内，用json去建立条件的相等关系

```java
//> db.car_info.count()
long count = mongoTemplate.count(new Query(), CarInfo.class);

//带条件

//> db.car_info.count({"car_id" : "京H55555"})
long count = mongoTemplate.count(new BasicQuery("{\"car_id\" : \"京H55555\"}"), CarInfo.class);
```
## 消除重复数据

语法如下

```
db.collection.distinct(field, query, options)
> db.student.distinct("name",{})
> db.student.distinct("name")
```

方式一：

```java

//> db.student.distinct("name",{})
		
List list = mongoTemplate.getCollection("car_info").distinct("car_id", new BasicDBObject());

```
方式二：

```java
//> db.runCommand({"distinct":"car_info","key":"car_id"})
//{ "values" : [ "京H12345", "京H55555" ], "ok" : 1 }
CommandResult executeCommand = mongoTemplate.executeCommand("{\"distinct\":\"car_info\",\"key\":\"car_id\"}");
```



## 6. aggregate操作

1. 在整个聚合框架里面。要引用每行的数据就采用“$”＋　字段名称
2. 几个操作符可以同事使用
3. 方法里的名称使用的是实体类的名称不是数据库字段的名称

### 6-1、$group

在MongoDB中会将集合依据指定的key的不同进行分组操作，并且每一个组都会产生一个处理的文档结果

`{ $group: { _id: <expression>, <field1>: { <accumulator1> : <expression1> }, ... } }`
_id是分组条件，如果_id设置为null，就是将整个文档作为一个整体组，不分组统计

```
db.sales.aggregate(
    [
        {
        $group : {
            _id : null, 设置为null，不分组，整个文档就是一个整体
            totalPrice: { $sum: { $multiply: [ "$price", "$quantity" ] } },
                averageQuantity: { $avg: "$quantity" },
                count: { $sum: 1 }
                }
        }
    ]
)
```
```
 db.student.aggregate({
"$group":{
    _id:"$name", name字段作为分组条件
        totalAge:{$sum:"$age"} 对age字段求和
    }
})

{
"$group":
{
"_id":"name",
totalAge:{$sum:"$age"},
count:{$sum:1} $sum 1就是求记录个数
}
}
```
数组显示$push
```
db.student.aggregate({"$group":{"_id":"$name",first_score:{$first:"$score"},name:{"$push":"$name"},count:{$sum:1}}})
{ "_id" : "zhansan-9", "first_score" : 75, "name" : [ "zhansan-9" ], "count" : 1 }
{ "_id" : "zhansan-6", "first_score" : 90, "name" : [ "zhansan-6" ], "count" : 1 }
{ "_id" : "zhansan-5", "first_score" : 92, "name" : [ "zhansan-5" ], "count" : 1 }
{ "_id" : "zhansan-8", "first_score" : 81, "name" : [ "zhansan-8" ], "count" : 1 }
{ "_id" : "zhansan-1", "first_score" : 85, "name" : [ "zhansan-1", "zhansan-1" ], "count" : 2 }
{ "_id" : "zhansan-3", "first_score" : 88, "name" : [ "zhansan-3" ], "count" : 1 }
{ "_id" : "zhansan-2", "first_score" : 85, "name" : [ "zhansan-2" ], "count" : 1 }
{ "_id" : "zhansan-7", "first_score" : 99, "name" : [ "zhansan-7" ], "count" : 1 }
{ "_id" : "zhansan-4", "first_score" : 95, "name" : [ "zhansan-4" ], "count" : 1 }
```
数组去重$addToSet
```
> db.student.aggregate({"$group":{"_id":"$name",first_score:{$first:"$score"},name:{"$addToSet":"$name"},count:{$sum:1}}})
{ "_id" : "zhansan-9", "first_score" : 75, "name" : [ "zhansan-9" ], "count" : 1 }
{ "_id" : "zhansan-6", "first_score" : 90, "name" : [ "zhansan-6" ], "count" : 1 }
{ "_id" : "zhansan-5", "first_score" : 92, "name" : [ "zhansan-5" ], "count" : 1 }
{ "_id" : "zhansan-8", "first_score" : 81, "name" : [ "zhansan-8" ], "count" : 1 }
{ "_id" : "zhansan-1", "first_score" : 85, "name" : [ "zhansan-1" ], "count" : 2 }
{ "_id" : "zhansan-3", "first_score" : 88, "name" : [ "zhansan-3" ], "count" : 1 }
{ "_id" : "zhansan-2", "first_score" : 85, "name" : [ "zhansan-2" ], "count" : 1 }
{ "_id" : "zhansan-7", "first_score" : 99, "name" : [ "zhansan-7" ], "count" : 1 }
{ "_id" : "zhansan-4", "first_score" : 95, "name" : [ "zhansan-4" ], "count" : 1 }
```

注意，所有的分组都是无序的





范例一：
```java
//使用name字段分组，求每组个数，平均值，总和，最大值，最小值
//db.user.aggregate({"$group":{"_id":"$name","count":{"$sum":1},"avg_age":{"$avg":"$age"},"total_age":{"$sum":"$age"},"max_age":{"$max":"$age"},"min_age":{"$min":"$age"}}})


AggregationResults<BasicDBObject> aggregate = mongoTemplate.aggregate(
        Aggregation.newAggregation(
                Aggregation.group("name").count().as("count").sum("age").as("total_age")
        ),User.class, BasicDBObject.class);
System.err.println(aggregate.getMappedResults());
```
范例二：
```java
//使用name分组，获取第一个年龄和所有的年龄
		//> db.user.aggregate([{"$group":{"_id":"$name","first_age":{"$first":"$age"},"all_age":{"$push":"$age"}}}])

		AggregationResults<BasicDBObject> aggregate = mongoTemplate.aggregate(
				new TypedAggregation<>(User.class,Aggregation.group("name").first("age").as("first_age").push("age").as("all_age")), 
				BasicDBObject.class);
		
		System.err.println(aggregate.getMappedResults());
        //结果如下：分组统计同时也可以用数组显示某一个字段在这个组内的所有的值的集合，如果要对集合去重，使用$addToSet
		[
            { "_id" : "wangwu" , "first_age" : 14.0 , "all_age" : [ 14.0 , 15.0 , 16.0]},
            { "_id" : "lisi" , "first_age" : 114.0 , "all_age" : [ 114.0]}, 
            { "_id" : "zhansan" , "first_age" : 12.0 , "all_age" : [ 12.0 , 113.0 , 114.0]}
         ]
```

### 6-2、$project


https://docs.spring.io/spring-data/data-mongodb/docs/2.0.2.RELEASE/reference/html/#new-features.1-7-0
```
a == b
{ $eq : [$a, $b] }
a != b
{ $ne : [$a , $b] }
a > b
{ $gt : [$a, $b] }
a >= b
{ $gte : [$a, $b] }
a < b
{ $lt : [$a, $b] }
a ⇐ b
{ $lte : [$a, $b] }
a + b
{ $add : [$a, $b] }
a - b
{ $subtract : [$a, $b] }
a * b
{ $multiply : [$a, $b] }
a / b
{ $divide : [$a, $b] }
a^b
{ $pow : [$a, $b] }
a % b
{ $mod : [$a, $b] }
a && b
{ $and : [$a, $b] }
a || b
{ $or : [$a, $b] }
!a
{ $not : [$a] }
```
project可以用来控制列的显示规则，相当于结果集的投影功能

1、普通列（{“成员”：1|true}）表示要显示的内容
2、_id列（{“_id”: 0|false}）表示“_id”列是否显示
3、条件过滤列（{“成员”：表达式}）满足表达式之后的数据可以进行显示


```java
//name字段显示，_id字段不显示，别名age是原始age字段×１４，别名hello标记原始age字段是否大于等于２０
// db.user.aggregate({"$project":{"name":1,"_id":0,"age":{"$multiply":["$age",14]},"hello":{"$gte":["$age",20]}}})


AggregationResults<BasicDBObject> aggregate = mongoTemplate.aggregate(
        Aggregation.newAggregation(
                Aggregation.project("name")
                .andExclude("_id")
                .andExpression("age * 14").as("age")
                .andExpression("age >= 20").as("hello")),
        User.class, BasicDBObject.class);
System.err.println(aggregate.getMappedResults());
```


### 6-3. $match

相当于where字句

```java
//age>=100的,只显示name和age,不显示_id
		//> db.user.aggregate([{"$project":{"name":1,"age":1,"_id":0}},{"$match":{"age":{"$gte":100}}}])
		AggregationResults<BasicDBObject> aggregate = mongoTemplate.aggregate(
				Aggregation.newAggregation(
						Aggregation.project("name","age").andExclude("_id"),
						Aggregation.match(Criteria.where("age").gte(100))), User.class, BasicDBObject.class);
		System.err.println(aggregate.getMappedResults());
```

### 6-4. $sort


1：升序　-1: 降序

```java

db.user.aggregate([{"$sort":{"age":-1}}])

// db.user.aggregate([{"$sort":{"age":-1}}])
		//age>=100的,只显示name和age,不显示_id
		//> db.user.aggregate([{"$project":{"name":1,"age":1,"_id":0}},{"$match":{"age":{"$gte":100}}},{"$sort":{"age":-1}}])
		AggregationResults<BasicDBObject> aggregate = mongoTemplate.aggregate(
				Aggregation.newAggregation(
						Aggregation.sort(Sort.Direction.DESC, "age"),
						Aggregation.project("name","age").andExclude("_id"),
						Aggregation.match(Criteria.where("age").gte(100))), User.class, BasicDBObject.class);
		System.err.println(aggregate.getMappedResults());

```

### 6-5. 分页

$limit:负责数据的取出个数
$skip:负责数据的跨过个数



```java
// db.user.aggregate([{"$skip":1},{"$limit":2}])
		// db.user.aggregate([{"$sort":{"age":-1}}])
		//age>=100的,只显示name和age,不显示_id
		//> db.user.aggregate([{"$project":{"name":1,"age":1,"_id":0}},{"$match":{"age":{"$gte":100}}},{"$sort":{"age":-1}}])
		AggregationResults<BasicDBObject> aggregate = mongoTemplate.aggregate(
				Aggregation.newAggregation(
						
						new SkipOperation(1),Aggregation.limit(2),
						Aggregation.sort(Sort.Direction.DESC, "age"),
						Aggregation.project("name","age").andExclude("_id"),
						Aggregation.match(Criteria.where("age").gte(100))), User.class, BasicDBObject.class);
		System.err.println(aggregate.getMappedResults());
		
```

### 6-6. $unwind


在查询数据的时候，经常返回数组信息，
这个操作付的所用就是将数组信息转换成字符串

```
{ "_id" : 1, "item" : "ABC1", sizes: [ "S", "M", "L"] }

db.inventory.aggregate( [ { $unwind : "$sizes" } ] )

{ "_id" : 1, "item" : "ABC1", "sizes" : "S" }
{ "_id" : 1, "item" : "ABC1", "sizes" : "M" }
{ "_id" : 1, "item" : "ABC1", "sizes" : "L" }
```

```java
Aggregation.unwind("sizes")
```
### 6-7. $geoNear

计算距离，必须要有索引的支持

```java
//		db.places.aggregate([
//		{
//		$geoNear: {
//		near: { type: "Point", coordinates: [ -73.99279 , 40.719296 ] },目标位置
//		distanceField: "loc",距离字段
//		maxDistance: 2,最远距离
//		query: { "hello": "public" },
//		includeLocs: "dist.location",
//		num: 5,返回条数
//		spherical: true球面方式
//		}
//		}
//		])

		Point location = new Point(-73.99171, 40.738868);
		NearQuery query = NearQuery.near(location)
				.maxDistance(new Distance(10, Metrics.MILES))
				.num(5)
				.spherical(true)
				.query(new BasicQuery("{ \"hello\": \"public\" }"));
	
		AggregationResults<BasicDBObject> aggregate = mongoTemplate.aggregate(
				Aggregation.newAggregation(
						Aggregation.geoNear(query, "loc")
						), User.class, BasicDBObject.class);
		System.err.println(aggregate.getMappedResults());
```


### 6-8、$out

将查询结果输出到指定的集合里面（表的复制操作，生成一张新表）
```java

	
//		db.books.aggregate( [
//		{ $group : { _id : "$author", books: { $push: "$title" } } },
//		{ $out : "authors" }
//		] )
		
	
		AggregationResults<BasicDBObject> aggregate = mongoTemplate.aggregate(
				Aggregation.newAggregation(
						Aggregation.group("author").push("title").as("books"),
						Aggregation.out("authors")
						), User.class, BasicDBObject.class);
		System.err.println(aggregate.getMappedResults());
```








