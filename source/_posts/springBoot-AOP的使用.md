---
title: springBoot-AOP的使用
date: 2017-07-03 21:23:11
updated: 2017-07-06 19:04:10
categories: 
- 后端
tags:
- Java
- Spring
---

# 添加aop starter依赖

```xml

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
 
	<groupId>com.hhbbz.springboot.aop</groupId>
	<artifactId>springboot-aop</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.1.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
 
	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
	</properties>
 
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		
		<dependency><!-- spring boot aop starter依赖 -->
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-aop</artifactId>
		</dependency>
 
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		
		<!--druid -->
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>druid</artifactId>
			<version>1.0.27</version>
		</dependency>
	</dependencies>
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>
```

# 配置文件中开启支持

```properties
spring.aop.auto=true
```

# 编写切面

```java
package com.hhbbz.springboot.aop.aop;
 
import java.util.Arrays;
 
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
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;
 
import com.alibaba.druid.support.json.JSONUtils;
 
@Component // 注册到Spring容器，必须加入这个注解
@Aspect // 该注解标示该类为切面类，切面是由通知和切点组成的。
public class ApiAspect {
	
	private static final Logger logger =  LoggerFactory.getLogger(ApiAspect.class);
 
    ThreadLocal<Long> startTime = new ThreadLocal<Long>();
 
	@Pointcut("execution(* com.hhbbz.springboot.aop.controller.HelloController.*(..))")// 定义切点表达式
	@Order(2)
	public void controllerPoint() {
	}
	
	@Pointcut("@annotation(com.hhbbz.springboot.aop.annotation.RedisCache)")// 定义注解类型的切点，只要方法上有该注解，都会匹配
	@Order(1)
	public void annotationPoint(){
		
	}
 
	// 定义前置通知
	@Before("annotationPoint()")
	public void before(JoinPoint joinPoint) {
		System.out.println("方法执行前执行.....before");
		ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();
        logger.info("<====================================================================");
        logger.info("请求来源:  => " + request.getRemoteAddr());
        logger.info("请求URL: " + request.getRequestURL().toString());
        logger.info("请求方式: " + request.getMethod());
        logger.info("响应方法: " + joinPoint.getSignature().getDeclaringTypeName() + "." + joinPoint.getSignature().getName());
        logger.info("请求参数 : " + Arrays.toString(joinPoint.getArgs()));
        logger.info("---------------------------------------------------------------------");
        startTime.set(System.currentTimeMillis());
	}
 
	@Around("controllerPoint() && args(arg)")// 需要匹配切点表达式，同时需要匹配参数
	public String around(ProceedingJoinPoint pjp, String arg) {
		System.out.println("name:"+arg);
		System.out.println("方法环绕start....around.");
		String result = null;
		try {
			result = (String) pjp.proceed()+" aop String";
		} catch (Throwable e) {
			e.printStackTrace();
		}
		System.out.println("方法环绕end.....around");
		return result;
	}
 
	@After("within(com.hhbbz.springboot.aop.controller.*Controller)")
	public void after() {
		System.out.println("方法之后执行....after.");
	}
 
	@AfterReturning(pointcut="controllerPoint()", returning="rst")
	public void afterReturning(JoinPoint joinPoint, Object rst) {
		System.out.println("方法执行完执行.....afterReturning");
		logger.info("耗时（毫秒） : " + (System.currentTimeMillis() - startTime.get()));
        logger.info("返回数据: {}", JSONUtils.toJSONString(rst));
        logger.info("====================================================================>");
	}
 
	@AfterThrowing("within(com.hhbbz.springboot.aop.controller.*Controller)")
	public void afterThrowing() {
		System.out.println("异常出现之后.....afterThrowing");
	}
}
```

# 定义注解

```java
package com.hhbbz.springboot.aop.annotation;
 
import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
 
/**
 * 描述：加入查询结果的缓存
 * @author hhbbz
 * 创建时间：2017年7月1日 上午10:30:00
 * @version 1.2.0
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Documented
public @interface RedisCache {
    Class<?> type();//被代理类的全类名，在之后会做为redis hash 的key
}
```

# 编写Controller

```java
package com.hhbbz.springboot.aop.controller;
 
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
 
import com.hhbbz.springboot.aop.annotation.RedisCache;
 
@RestController
public class HelloController {
 
  @RequestMapping(value = "/hello")
  public String hello(@RequestParam(value = "name", required = true) String name) {
    String result = "hello  " + name;
    System.out.println(result);
    return result;
  }
  
  @RequestMapping(value = "/world")
  public String world(@RequestParam(value = "arg", required = true) String arg){
	  String result = "world  " + arg;
	    System.out.println(result);
	    return result;
  }
  
  @RequestMapping(value="/annotation")
  @RedisCache(type=String.class)
  public String annotation(){
	  return "annotation";
  }
}
```

# 测试

1. 启动服务

2. 在浏览器中输入：http://localhost:8080/annotation

3. 测试结果如下

```log
方法执行前执行.....before
2017-07-06 18:40:23.273  INFO 2208 --- [nio-8080-exec-1] com.hhbbz.springboot.aop.aop.ApiAspect  : <====================================================================
2017-07-06 18:40:23.274  INFO 2208 --- [nio-8080-exec-1] com.hhbbz.springboot.aop.aop.ApiAspect  : 请求来源:  => 127.0.0.1
2017-07-06 18:40:23.274  INFO 2208 --- [nio-8080-exec-1] com.hhbbz.springboot.aop.aop.ApiAspect  : 请求URL: http://localhost:8080/annotation
2017-07-06 18:40:23.274  INFO 2208 --- [nio-8080-exec-1] com.hhbbz.springboot.aop.aop.ApiAspect  : 请求方式: GET
2017-07-06 18:40:23.276  INFO 2208 --- [nio-8080-exec-1] com.hhbbz.springboot.aop.aop.ApiAspect  : 响应方法: com.hhbbz.springboot.aop.controller.HelloController.annotation
2017-07-06 18:40:23.276  INFO 2208 --- [nio-8080-exec-1] com.hhbbz.springboot.aop.aop.ApiAspect  : 请求参数 : []
2017-07-06 18:40:23.276  INFO 2208 --- [nio-8080-exec-1] com.hhbbz.springboot.aop.aop.ApiAspect  : ---------------------------------------------------------------------
方法之后执行....after.
方法执行完执行.....afterReturning
2017-07-06 18:40:23.286  INFO 2208 --- [nio-8080-exec-1] com.hhbbz.springboot.aop.aop.ApiAspect  : 耗时（毫秒） : 10
2017-07-06 18:40:23.289  INFO 2208 --- [nio-8080-exec-1] com.hhbbz.springboot.aop.aop.ApiAspect  : 返回数据: "annotation"
2017-07-06 18:40:23.291  INFO 2208 --- [nio-8080-exec-1] com.hhbbz.springboot.aop.aop.ApiAspect  : ====================================================================>
```

由于Around环绕通知不匹配，所以没有切入进去.