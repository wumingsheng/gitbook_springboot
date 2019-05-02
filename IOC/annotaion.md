# 注解IOC

* @Component:组件:

	有三个衍生注解:

	* @Controller    :描述Web层的类
	* @Service        :描述业务层的类
	* @Repository    :描述持久层的类

* @Value:注入普通类型属性(字符串和基本数据类型)

* @Autowired:自动装配(此注解是按照类型自动装配的),如果需要指定名称,加@Qualifier( "" )

    * **@Resource注解:相当于: @Autowired + @Qulifer(value=””)**

* @Scope(value="prototype"):Spring生成类的过程中多例的模式,指定bean的作用范围

* @PostConstruct:相当于init-method,Bean初始化的时候执行的方法

* @PreDestroy:相当于destroy-method,Bean销毁的时候执行的方法(1. 对象必须是单例创建的才可以销毁.2. 必须等到工厂关闭的时候才会销毁对象.)