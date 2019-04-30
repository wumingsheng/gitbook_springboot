# springboot整合prometheus(二)

springboot整合prometheus有两种方式：

1. 通过官方提供的依赖包`simpleclient_spring_boot`,但是好像对于`springboot2.X`不是很给力，对于`springboot2.X`我们可以使用一个桥接包`compile group: 'io.micrometer', name: 'micrometer-registry-prometheus', version: '1.0.6'`

2. 其实官方给的整合包`simpleclient_spring_boot`也是在自己原生方式上作了一个封装，方便大家使用，但是由于更新不即使，带来了使用上的不够灵活，因此下面先体验一下原生的方式


整个过程大致可以分为三个步骤：

**收集Collector → 注册Register → 暴露Exporting**





