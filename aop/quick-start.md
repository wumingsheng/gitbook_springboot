# 快速开始


## 依赖

```groovy
implementation 'org.springframework.boot:spring-boot-starter-aop'
compile group: 'org.apache.commons', name: 'commons-lang3', version: '3.9'
compile group: 'org.javassist', name: 'javassist', version: '3.25.0-GA'
compile group: 'com.google.code.gson', name: 'gson', version: '2.8.5'
```

## service切面

service切面主要是记录参数和返回值

```java

import org.apache.commons.lang3.ArrayUtils;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.AfterThrowing;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.context.annotation.EnableAspectJAutoProxy;
import org.springframework.stereotype.Component;

import com.google.gson.Gson;

import javassist.ClassClassPath;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;
import javassist.Modifier;
import javassist.bytecode.CodeAttribute;
import javassist.bytecode.LocalVariableAttribute;
import javassist.bytecode.MethodInfo;
import lombok.extern.slf4j.Slf4j;

@Component
@Aspect
@Slf4j
@EnableAspectJAutoProxy
public class ServiceAspect {

	@Pointcut("execution(* com.example.asyncdemo.service..*.*(..))")
	public void excuteService() {
	}
	
	@After(value = "excuteService()")
	public void after() {
		log.info("===========最终final通知，不管是否异常，该通知都会执行");
	}

	@AfterThrowing(value = "excuteService()", throwing = "ex")
	public void afterThrowing(Throwable ex) {
		System.out.println("异常抛出通知==============" + ex.getMessage());
	}

	@Around(value = "excuteService()")
	public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
		System.out.println("环绕前通知=======开始时间:" + System.currentTimeMillis());
		Object obj = joinPoint.proceed();
		System.out.println("环绕后通知=======结束时间:" + System.currentTimeMillis());
		return obj;
	}

	@AfterReturning(value = "excuteService()", returning = "result")
	public void afterReturning(JoinPoint joinPoint, Object result) {
		String classType = joinPoint.getTarget().getClass().getName();
		String methodName = joinPoint.getSignature().getName();
		log.info("class_name = {}", classType);
		log.info("method_name = {}", methodName);
		Gson gson = new Gson();
		log.info("请求结束===返回值:" + gson.toJson(result));
	}

	@Before(value = "excuteService()")
	public void before(JoinPoint joinPoint) {
		String classType = joinPoint.getTarget().getClass().getName();
		String methodName = joinPoint.getSignature().getName();
		log.info("class_name = {}", classType);
		log.info("method_name = {}", methodName);

		Object[] args = joinPoint.getArgs();

		try {
			/**
			 * 获取方法参数名称
			 */
			String[] paramNames = getFieldsName(classType, methodName);

			/**
			 * 打印方法的参数名和参数值
			 */
			logParam(paramNames, args);
		} catch (Exception e) {
			e.printStackTrace();
		}

	}

	/**
	 * 判断是否为基本类型：包括String
	 * 
	 * @param clazz clazz
	 * @return true：是; false：不是
	 */
	private boolean isPrimite(Class<?> clazz) {
		if (clazz.isPrimitive() || clazz == String.class) {
			return true;
		} else {
			return false;
		}
	}

	/**
	 * 打印方法参数值 基本类型直接打印，非基本类型需要重写toString方法
	 * 
	 * @param paramsArgsName  方法参数名数组
	 * @param paramsArgsValue 方法参数值数组
	 */
	private void logParam(String[] paramsArgsName, Object[] paramsArgsValue) {
		if (ArrayUtils.isEmpty(paramsArgsName) || ArrayUtils.isEmpty(paramsArgsValue)) {
			log.info("该方法没有参数");
			return;
		}
		StringBuffer buffer = new StringBuffer();
		for (int i = 0; i < paramsArgsName.length; i++) {
			// 参数名
			String name = paramsArgsName[i];
			// 参数值
			Object value = paramsArgsValue[i];
			buffer.append(name + " = ");
			if (isPrimite(value.getClass())) {
				buffer.append(value + "  ,");
			} else {
				buffer.append(value.toString() + "  ,");
			}
		}
		log.info(buffer.toString());
	}

	/**
	 * 使用javassist来获取方法参数名称
	 * 
	 * @param class_name  类名
	 * @param method_name 方法名
	 * @return
	 * @throws Exception
	 */
	private String[] getFieldsName(String class_name, String method_name) throws Exception {
		Class<?> clazz = Class.forName(class_name);
		String clazz_name = clazz.getName();
		ClassPool pool = ClassPool.getDefault();
		ClassClassPath classPath = new ClassClassPath(clazz);
		pool.insertClassPath(classPath);

		CtClass ctClass = pool.get(clazz_name);
		CtMethod ctMethod = ctClass.getDeclaredMethod(method_name);
		MethodInfo methodInfo = ctMethod.getMethodInfo();
		CodeAttribute codeAttribute = methodInfo.getCodeAttribute();
		LocalVariableAttribute attr = (LocalVariableAttribute) codeAttribute.getAttribute(LocalVariableAttribute.tag);
		if (attr == null) {
			return null;
		}
		String[] paramsArgsName = new String[ctMethod.getParameterTypes().length];
		int pos = Modifier.isStatic(ctMethod.getModifiers()) ? 0 : 1;
		for (int i = 0; i < paramsArgsName.length; i++) {
			paramsArgsName[i] = attr.variableName(i + pos);
		}
		return paramsArgsName;
	}

}

```

## controller切面

controller切面可以在request对象上做一些剖析记录

```java

import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;

import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.AfterThrowing;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.context.annotation.EnableAspectJAutoProxy;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestAttributes;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import com.google.gson.Gson;

import lombok.extern.slf4j.Slf4j;

@Component
@Aspect
@Slf4j
@EnableAspectJAutoProxy
public class ControllerAspect {

	@Pointcut("execution(* com.example.asyncdemo.controller..*.*(..))")
	public void excuteController() {
	}
	
	@After(value = "excuteController()")
	public void after() {
		log.info("===========最终final通知，不管是否异常，该通知都会执行");
	}

	@AfterThrowing(value = "excuteController()", throwing = "ex")
	public void afterThrowing(Throwable ex) {
		System.out.println("异常抛出通知==============" + ex.getMessage());
	}

	@Around(value = "excuteController()")
	public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
		System.out.println("环绕前通知=======开始时间:" + System.currentTimeMillis());
		Object obj = joinPoint.proceed();
		System.out.println("环绕后通知=======结束时间:" + System.currentTimeMillis());
		return obj;
	}

	@AfterReturning(value = "excuteController()", returning = "result")
	public void afterReturning(JoinPoint joinPoint, Object result) {
		String classType = joinPoint.getTarget().getClass().getName();
		String methodName = joinPoint.getSignature().getName();
		log.info("class_name = {}", classType);
		log.info("method_name = {}", methodName);
		Gson gson = new Gson();
		log.info("请求结束===返回值:" + gson.toJson(result));
	}

	@Before(value = "excuteController()")
	public void before(JoinPoint joinPoint) {
		String classType = joinPoint.getTarget().getClass().getName();
		String methodName = joinPoint.getSignature().getName();
		log.info("class_name = {}", classType);
		log.info("method_name = {}", methodName);

		Object[] args = joinPoint.getArgs();
		
		 RequestAttributes ra = RequestContextHolder.getRequestAttributes();
	        ServletRequestAttributes sra = (ServletRequestAttributes) ra;
	        HttpServletRequest request = sra.getRequest();
	        String method = request.getMethod();
	        String uri = request.getRequestURI();
	        String queryString = request.getQueryString();
	        String params = "";
	        
	        //获取请求参数集合并进行遍历拼接
	        if(args.length>0){
	            if("POST".equals(method)){
	                Object object = args[0];
	                Map<String, Object> map = getKeyAndValue(object);
	                Gson gson = new Gson();
	               
	                params =  gson.toJson(map)
	;
	            }else if("GET".equals(method)){
	                params = queryString;
	            }
	        }
	 
	 
	        log.info("请求开始===地址:"+uri);
	        log.info("请求开始===类型:"+method);
	        log.info("请求开始===参数:"+params);




	
	}
	private  Map<String, Object> getKeyAndValue(Object obj) {
        Map<String, Object> map = new HashMap<>();
        // 得到类对象
        Class<? extends Object> userCla = (Class<? extends Object>) obj.getClass();
        /* 得到类中的所有属性集合 */
        Field[] fs = userCla.getDeclaredFields();
        for (int i = 0; i < fs.length; i++) {
            Field f = fs[i];
            f.setAccessible(true); // 设置些属性是可以访问的
            Object val = new Object();
            try {
                val = f.get(obj);
                // 得到此属性的值
                map.put(f.getName(), val);// 设置键值
            } catch (IllegalArgumentException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
 
            }
        return map;
    }


	

}

```












