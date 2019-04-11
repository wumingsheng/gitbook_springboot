# logstash-json日志

springboot和logstash整合，可以直接将日志输入到logstash的服务端上，也可以将日志以json格式输出到制定位置的文件，由logstash自己去收集

下面先介绍springboot输出标准logstash的本地json格式文件

## logback输出标准logstash的本地json格式文件

### 1. 需要用到的依赖包pom.xml

```xml
<dependency>
	<groupId>net.logstash.logback</groupId>
	<artifactId>logstash-logback-encoder</artifactId>
	<version>5.1</version>
</dependency>
```

### 2. springboot配置application.yml

```yml

spring:
  application:
    name: service-demo           
management:
  endpoints:
    web:
      exposure:
        include: "*"
logging:
  root: /opt/service/log
  config: classpath:service-logback.xml
  file: /opt/service/log/${spring.application.name}/${spring.application.name}.info.log

```
### 3. logback配置文件logback.xml

```xml
 <!-- 为logstash输出的JSON格式的Appender -->
   <appender name="logstash" class="ch.qos.logback.core.rolling.RollingFileAppender">
   		<file>${LOG_PATH}/${APP_NAME}/${APP_NAME}.info-logstash.json</file>
	    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
	        <fileNamePattern>${LOG_PATH}/${APP_NAME}/${APP_NAME}.%d{yyyy-MM-dd}.info-logstash.%i.json</fileNamePattern>
	        <maxHistory>30</maxHistory>
	        <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
				<maxFileSize>512MB</maxFileSize>
			</timeBasedFileNamingAndTriggeringPolicy>
	    </rollingPolicy>
	    <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder" charset="UTF-8">
	      <providers>
	        <timestamp>
	            <timeZone>UTC</timeZone>
	        </timestamp>
	        <pattern>
	            <pattern>
		            {
			            "app": "${APP_NAME}",
			            "level": "%level",
			            "pid": "${PID:-}",
			            "thread": "%thread",
			            "class": "%logger{40}",
			            "message": "%message",
			            "stack_trace": "%exception{10}"
		            }
	            </pattern>
	        </pattern>
	      </providers>
	    </encoder>
  </appender>


	<root level="INFO">
		<appender-ref ref="logstash" />
	</root>
```

### 4. 输出

```json
{"@timestamp":"2019-04-11T07:07:39.334+00:00","app":"service-demo","level":"INFO","pid":"8699","thread":"restartedMain","class":"com.example.demo.DemoApplication","message":"Starting DemoApplication on user-PC with PID 8699 (/home/user/Documents/code-source/demo/target/classes started by user in /home/user/Documents/code-source/demo)","stack_trace":""}
{"@timestamp":"2019-04-11T07:07:39.338+00:00","app":"service-demo","level":"INFO","pid":"8699","thread":"restartedMain","class":"com.example.demo.DemoApplication","message":"No active profile set, falling back to default profiles: default","stack_trace":""}
{"@timestamp":"2019-04-11T07:07:39.357+00:00","app":"service-demo","level":"INFO","pid":"8699","thread":"restartedMain","class":"o.s.b.d.e.DevToolsPropertyDefaultsPostProcessor","message":"Devtools property defaults active! Set 'spring.devtools.add-properties' to 'false' to disable","stack_trace":""}
{"@timestamp":"2019-04-11T07:07:39.358+00:00","app":"service-demo","level":"INFO","pid":"8699","thread":"restartedMain","class":"o.s.b.d.e.DevToolsPropertyDefaultsPostProcessor","message":"For additional web related logging consider setting the 'logging.level.web' property to 'DEBUG'","stack_trace":""}
{"@timestamp":"2019-04-11T07:07:40.046+00:00","app":"service-demo","level":"INFO","pid":"8699","thread":"restartedMain","class":"o.s.b.w.embedded.tomcat.TomcatWebServer","message":"Tomcat initialized with port(s): 8080 (http)","stack_trace":""}
{"@timestamp":"2019-04-11T07:07:40.063+00:00","app":"service-demo","level":"INFO","pid":"8699","thread":"restartedMain","class":"org.apache.catalina.core.StandardService","message":"Starting service [Tomcat]","stack_trace":""}
{"@timestamp":"2019-04-11T07:07:40.064+00:00","app":"service-demo","level":"INFO","pid":"8699","thread":"restartedMain","class":"org.apache.catalina.core.StandardEngine","message":"Starting Servlet engine: [Apache Tomcat/9.0.17]","stack_trace":""}
{"@timestamp":"2019-04-11T07:07:40.104+00:00","app":"service-demo","level":"INFO","pid":"8699","thread":"restartedMain","class":"o.a.c.c.C.[Tomcat].[localhost].[/]","message":"Initializing Spring embedded WebApplicationContext","stack_trace":""}
{"@timestamp":"2019-04-11T07:07:40.104+00:00","app":"service-demo","level":"INFO","pid":"8699","thread":"restartedMain","class":"o.s.web.context.ContextLoader","message":"Root WebApplicationContext: initialization completed in 746 ms","stack_trace":""}
{"@timestamp":"2019-04-11T07:07:40.371+00:00","app":"service-demo","level":"INFO","pid":"8699","thread":"restartedMain","class":"o.s.s.concurrent.ThreadPoolTaskExecutor","message":"Initializing ExecutorService 'applicationTaskExecutor'","stack_trace":""}
{"@timestamp":"2019-04-11T07:07:40.528+00:00","app":"service-demo","level":"INFO","pid":"8699","thread":"restartedMain","class":"o.s.b.d.a.OptionalLiveReloadServer","message":"LiveReload server is running on port 35729","stack_trace":""}
{"@timestamp":"2019-04-11T07:07:40.534+00:00","app":"service-demo","level":"INFO","pid":"8699","thread":"restartedMain","class":"o.s.b.a.e.web.EndpointLinksResolver","message":"Exposing 16 endpoint(s) beneath base path '/actuator'","stack_trace":""}
{"@timestamp":"2019-04-11T07:07:40.594+00:00","app":"service-demo","level":"INFO","pid":"8699","thread":"restartedMain","class":"o.s.b.w.embedded.tomcat.TomcatWebServer","message":"Tomcat started on port(s): 8080 (http) with context path ''","stack_trace":""}
{"@timestamp":"2019-04-11T07:07:40.599+00:00","app":"service-demo","level":"INFO","pid":"8699","thread":"restartedMain","class":"com.example.demo.DemoApplication","message":"Started DemoApplication in 1.58 seconds (JVM running for 1.992)","stack_trace":""}
{"@timestamp":"2019-04-11T07:07:40.610+00:00","app":"service-demo","level":"INFO","pid":"8699","thread":"restartedMain","class":"com.example.demo.DemoApplication","message":"开始记录日志。。。","stack_trace":""}

```







