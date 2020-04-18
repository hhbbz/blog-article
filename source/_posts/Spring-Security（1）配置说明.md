---
title: Spring Security（1）配置说明
date: 2018-07-06 18:35:40
categories: 
- 后端
tags:
- Java
- Spring
---

# Spring Security 模块

- **核心模块** - spring-security-core.jar：包含核心验证和访问控制类和接口，远程支 持的基本配置API，是基本模块

- **远程调用** - spring-security-remoting.jar：提供与 Spring Remoting 集成

- **网页** - spring-security-web.jar：包括网站安全的模块，提供网站认证服务和基于URL访问控制
- **配置** - spring-security-config.jar：包含安全命令空间解析代码，若使用XML进行配置则需要
- **LDAP** - spring-security-ldap.jar：LDAP 验证和配置，若需要LDAP验证和管理LDAP用户实体
- **ACL访问控制表** - spring-security-acl.jar：ACL专门领域对象的实现
- **CAS** - spring-security-cas.jar：CAS客户端继承，若想用CAS的SSO服务器网页验证
- **OpenID** - spring-security-openid.jar：OpenID网页验证支持
- **Test** - spring-security-test.jar：支持Spring Security的测试

# 默认执行顺序

## UsernamePasswordAuthenticationFilter

1. 用户通过url：/login 登录，该过滤器接收表单用户名密码
2. 判断用户名密码是否为空
3. 生成 UsernamePasswordAuthenticationToken
4. 将Authentiction 传给 AuthenticationManager接口的 authenticate 方法进行认证处理
5. AuthenticationManager 默认是实现类为 ProviderManager ，ProviderManager 委托给 AuthenticationProvider 进行处理
6. UsernamePasswordAuthenticationFilter 继承了 AbstractAuthenticationProcessingFilter 抽象类，AbstractAuthenticationProcessingFilter 在 successfulAuthentication 方法中对登录成功进行了处理，通过 SecurityContextHolder.getContext().setAuthentication() 方法将 Authentication 认证信息对象绑定到 SecurityContext
7. 下次请求时，在过滤器链头的 SecurityContextPersistenceFilter 会从 Session 中取出用户信息并生成 Authentication（默认为 UsernamePasswordAuthenticationToken），并通过 SecurityContextHolder.getContext().setAuthentication() 方法将 Authentication 认证信息对象绑定到 SecurityContext
8. 需要权限才能访问的请求会从 SecurityContext 中获取用户的权限进行验证
DaoAuthenticationProvider （实现了 AuthenticationProvider）：
通过 UserDetailsService 获取 UserDetails
将 UserDetails 和 UsernamePasswordAuthentionToken 进行认证匹配用户名密码是否正确
若正确则通过 UserDetails 中用户的权限、用户名等信息生成一个新的 Authentication 认证对象并返回

## DaoAuthenticationProvider （实现了 AuthenticationProvider）

1. 通过 UserDetailsService 获取 UserDetails
2. 将 UserDetails 和 UsernamePasswordAuthentionToken 进行认证匹配用户名密码是否正确
3. 若正确则通过 UserDetails 中用户的权限、用户名等信息生成一个新的 Authentication 认证对象并返回

# 相关类

## WebSecurityConfigurerAdapter

- 为创建 WebSecurityConfigurer 实例提供方便的基类，重写它的 configure 方法来设置安全细节
    - configure(HttpSecurity http)：重写该方法，通过 http 对象的 authorizeRequests()方法定义URL访问权限，默认会为 formLogin() 提供一个简单的测试HTML页面
    - _configureGlobal(AuthenticationManagerBuilder auth) _：通过 auth 对象的方法添加身份验证

## SecurityContextHolder

- SecurityContextHolder 用于存储安全上下文信息（如操作用户是谁、用户是否被认证、用户权限有哪些），它用 ThreadLocal 来保存 SecurityContext，者意味着 Spring Security 在用户登录时自动绑定到当前现场，用户退出时，自动清除当前线程认证信息，SecurityContext 中含有正在访问系统用户的详细信息

## AuthenticationManagerBuilder

- 用于构建认证 AuthenticationManager 认证，允许快速构建内存认证、LDAP身份认证、JDBC身份验证，添加 userDetailsService（获取认证信息数据） 和 AuthenticationProvider's（定义认证方式）
- 内存验证：
```java
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
      auth.inMemoryAuthentication()
            .withUser("user").password("123").roles("USER").and()
            .withUser("admin").password("456").roles("USER","ADMIN");
}
```
JDBC 验证：
```java
@Autowired
private DataSource dataSource;

@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth.jdbcAuthentication()
            .dataSource(dataSource)
            .withDefaultSchema()
            .withUser("linyuan").password("123").roles("USER").and()
            .withUser("linyuan2").password("456").roles("ADMIN");
}
```

- userDetailsService(T userDetailsService)：传入自定义的 UserDetailsService 获取认证信息数据
- authenticationProvider(AuthenticationProvider authenticationProvider) ：传入自定义认证过程

## UserDetailsService

- 该接口仅有一个方法 loadUserByUsername，Spring Security 通过该方法获取.可自定义。

## UserDetails

- 我们可以实现该接口来定义自己认证用户的获取方式（如数据库中获取），认证成功后会将 UserDetails 赋给 Authentication 的 principal 属性，然后再把 Authentication 保存到 SecurityContext 中，之后需要实用用户信息时通过 SecurityContextHolder 获取存放在 SecurityContext 中的 Authentication 的 principal。

## Authentication

- Authentication 是一个接口，用来表示用户认证信息，在用户登录认证之前相关信息（用户传过来的用户名密码）会封装为一个 Authentication 具体实现类对象，默认情况下为 UsernamePasswordAuthenticationToken，登录之后（通过AuthenticationManager认证）会生成一个更详细的、包含权限的对象，然后把它保存在权限线程本地的 SecurityContext 中，供后续权限鉴定用

## GrantedAuthority

- GrantedAuthority 是一个接口，它定义了一个 getAuthorities() 方法返回当前 Authentication 对象的拥有权限字符串，用户有权限是一个 GrantedAuthority 数组，每一个 GrantedAuthority 对象代表一种用户
- 通常搭配 SimpleGrantedAuthority 类使用

## AuthenticationManager

- AuthenticationManager 是用来处理认证请求的接口，它只有一个方法 authenticate()，该方法接收一个 Authentication 作为对象，如果认证成功则返回一个封装了当前用户权限信息的 Authentication 对象进行返回
- 它默认的实现是 ProviderManager，但它不处理认证请求，而是将委托给 AuthenticationProvider 列表，然后依次使用 AuthenticationProvider 进行认证，如果有一个 AuthenticationProvider 认证的结果不为null，则表示成功（否则失败，抛出 ProviderNotFoundException），之后不在进行其它 AuthenticationProvider 认证，并作为结果保存在 ProviderManager
- 认证校验时最常用的方式就是通过用户名加载 UserDetails，然后比较 UserDetails 密码与请求认证是否一致，一致则通过，Security 内部的 DaoAuthenticationProvider 就是实用这种方式
- 认证成功后加载 UserDetails 来封装要返回的 Authentication 对象，加载的 UserDetails 对象是包含用户权限等信息的。认证成功返回的 Authentication 对象将会保存在当前的 SecurityContext 中

## AuthenticationProvide

- AuthenticationProvider 是一个身份认证接口，实现该接口来定制自己的认证方式，可通过 UserDetailsSevice 对获取数据库中的数据
- 该接口中有两个需要实现的方法：

  -  Authentication authenticate(Authentication authentication)：认证处理，返回一个 Authentication 的实现类则代表成功，返回 null 则为认证失败
  - supports(Class<?> aClass)：如果该 AuthenticationProvider 支持传入的 Authentication 认证对象，则返回 true ，如：return aClass.equals(UsernamePasswordAuthenticationToken.class);

## AuthorityUtils

- 是一个工具包，用于操作 GrantedAuthority 集合的实用方法：
    - commaSeparatedStringToAuthorityList(String authorityString)：从逗号分隔符中创建 GrantedAuthority 对象数组

## AbstractAuthenticationProcessingFilter

- 该抽象类继承了 GenericFilterBean，是处理 form 登录的过滤器，与 form 登录相关的所有操作都在该抽象类的子类中进行（UsernamePasswordAuthenticationFilter 为其子类），比如获取表单中的用户名、密码，然后进行认证等操作
- 该类在 doFilter 方法中通过 attemptAuthentication() 方法进行用户信息逻辑认证，认证成功会将用户信息设置到 Session 中

## UserDetails

- 代表了Spring Security的用户实体类，带有用户名、密码、权限特性等性质，可以自己实现该接口，供 Spring Security 安全认证使用，Spring Security 默认使用的是内置的 User 类
- 将从数据库获取的 User 对象传入实现该接口的类，并获取 User 对象的值来让类实现该接口的方法
- 通过 Authentication.getPrincipal() 的返回类型是 Object，但很多情况下其返回的其实是一个 UserDetails 的实例

## HttpSecurity

- 用于配置全局 Http 请求的权限控制规则，对哪些请求进行验证、不验证等
- 常用方法：
- authorizeRequests()：返回一个配置对象用于配置请求的访问限制
- formLogin()：返回表单配置对象，当什么都不指定时会提供一个默认的，如配置登录请求，还有登录成功页面
- logout()：返回登出配置对象，可通过logoutUrl设置退出url
- antMatchers：匹配请求路径或请求动作类型，如：.antMatchers("/admin/**")
- addFilterBefore: 在某过滤器之前添加 filter
- addFilterAfter：在某过滤器之后添加 filter
- addFilterAt：在某过滤器相同位置添加 filter，不会覆盖相同位置的 filter
- hasRole：结合 antMatchers 一起使用，设置请求允许访问的角色权限或IP，如：

```java
.antMatchers("/admin/**").hasAnyRole("ROLE_ADMIN","ROLE_USER")
```

方法名  |  用途
:-|:-|-:
access(String) |	SpringEL表达式结果为true时可访问
anonymous() |	匿名可访问
denyAll() |	用户不可以访问
fullyAuthenticated() |	用户完全认证访问（非remember me下自动登录）
hasAnyAuthority(String…) |	参数中任意权限可访问
hasAnyRole(String…) |	参数中任意角色可访问
hasAuthority(String) |	某一权限的用户可访问
hasRole(String) |	某一角色的用户可访问
permitAll()	 |所有用户可访问
rememberMe() |	允许通过remember me登录的用户访问
authenticated()	 |用户登录后可访问
hasIpAddress(String) |	用户来自参数中的IP可访问

## 注解与Spring EL

- **@EnableWebSecurity**：开启 Spring Security 注解
- **@EnableGlobalMethodSecurity(prePostEnabled=true)**：开启security方法注解
- **@PreAuthorize**：在方法调用前，通过SpringEL表达式限制方法访问

```java
@PreAuthorize("hasRole('ROLE_ADMIN')")
public void addUser(User user){
    //如果具有权限 ROLE_ADMIN 访问该方法
    ....
}
```

- **@PostAuthorize**：允许方法调用，但时如果表达式结果为false抛出异常

```java
//returnObject可以获取返回对象user，判断user属性username是否和访问该方法的用户对象的用户名一样。不一样则抛出异常。
@PostAuthorize("returnObject.user.username==principal.username")
public User getUser(int userId){
   //允许进入
...
    return user;
}
```

- **@PostFilter**：允许方法调用，但必须按表达式过滤方法结果

```java
//将结果过滤，即选出性别为男的用户
@PostFilter("returnObject.user.sex=='男' ")
public List<User> getUserList(){
    //允许进入
    ...
    return user;
}
```

- **@PreFilter**：允许方法调用，但必须在进入方法前过滤输入值

## Spring EL 表达式

表达式 | 描述
:-|:-|
hasRole ([role])|当前用户是否拥有指定角色
hasAnyRole([role1,role2])|多个角色是一个以逗号进行分隔的字符串。如果当前用户拥有指定角色中的任意一个则返回true
hasAuthority ([auth])|等同于hasRole
hasAnyAuthority([auth1,auth2])|等同于 hasAnyRole
Principle|代表当前用户的 principle对象
authentication|直接从 Security context获取的当前 Authentication对象
permitAll|总是返回true,表示允许所有的访问
denyAll|总是返回false,表示拒绝所有的访问访问
isAnonymous()|当前用户是否是一个匿名用户
isRememberMe|表示当前用户是否是通过remember - me自动登录的
isAuthenticated()|表示当前用户是否已经登录认证成功了
isFullAuthenticated()|如果当前用户既不是一个匿名用户,同时又不是通过Remember-Me自动登录的,则返回true

## 密码加密（PassWordEncoder）

- Spring 提供的一个用于对密码加密的接口，首选实现类为 BCryptPasswordEncoder
- 通过@Bean注解将它注入到IOC容器：

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

## 过滤器链

### SecurityContextPersistenceFilter

- 过滤器链头，是从 SecurityContextRepository 中取出用户认证信息，默认实现为 HttpSessionSecurityContextRepository，它会从 Session 中取出已认证的用户信息，提高效率，避免每次请求都要查询用户认证信息
- 取出之后会放入 SecurityContextHolder 中，以便其它 filter 使用，SecurityContextHolder 使用 ThreadLocal 存储用户认证信息，保证线程之间信息隔离，最后再 finally 中清除该信息

### WebAsyncManagerIntegrationFilter

- 提供了对 SecurityContext 和 WebAsyncManager 的集成，会把 SecurityContext 设置到异步线程，使其也能获取到用户上下文认证信息

### HanderWriterFilter

- 会往请求的 Header 中添加相应的信息

### CsrfFilter

- 跨域请求伪造过滤器，通过客户端穿来的 token 与服务端存储的 token 进行对比来判断请求

### LogoutFilter

- 匹配URL，默认为 /logout，匹配成功后则会用户退出，清除认证信息，若有自己的退出逻辑，该过滤器可以关闭

### UsernamePasswordAuthenticationFilter

- 登录认证过滤器，默认是对 /login 的 POST 请求进行认证，首先该方法会调用 attemptAuthentication 尝试认证获取一个 Authentication 认证对象，然后通过 sessionStrategy.onAuthentication 执行持久化，也就是保存认证信息，然后转向下一个 Filter，最后调用 successfulAuthentication 执行认证后事件
- attemptAuthentication 该方法是认证的主要方法，认证基本流程为 UserDeatilService 根据用户名获取到用户信息，然后通过 UserDetailsChecker.check 对用户状态进行校验，最后通过 additionalAuthenticationChecks 方法对用户密码进行校验完后认证后，返回一个认证对象

### DefaultLoginPageGeneratingFilter

- 当请求为登录请求时，生成简单的登录页面，可以关闭

### BasicAuthenticationFilter

- Http Basci 认证的支持，该认证会把用户名密码使用 base64 编码后放入 header 中传输，认证成功后会把用户信息放入 SecurityContextHolder 中

### RequestCacheAwareFilter

- 恢复被打断时的请求

### SecurityContextHolderAwareRequestFilter

- 针对 Servlet api 不同版本做一些包装

### AnonymousAuthenticationFIlter

- SecurityContextHolder 中认证信息为空，则会创建一个匿名用户到 SecurityContextHolder 中

### SessionManagementFilter

- 与登录认证拦截时作用一样，持久化用户登录信息，可以保存到 Session 中，也可以保存到 cookie 或 redis 中

### ExceptionTranslationFilter

- 异常拦截，处于 Filter 链条后部，只能拦截其后面的节点并着重处理 AuthenticationException 与 AccessDeniedException 异常

