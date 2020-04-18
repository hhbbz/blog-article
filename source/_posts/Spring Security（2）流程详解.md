---
title: Spring Security（2）流程详解
date: 2018-07-07 17:34:19
categories: 
- 后端
tags:
- Java
- Spring
---
# 简介

上章记录一点基础的配置，这次结合高级认证灵活使用Spring Security的用户认证。

# Spring Boot 添加 Spring Security

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

# 认证流程

1. 用户使用用户名和密码登录
2. 用户名密码被过滤器（默认为 UsernamePasswordAuthenticationFilter）获取到，配合其他权限信息（自定义），根据UsernamePasswordAuthenticationToken封装成一个未认证的Authentication（处在securityContext中）
3. Token（Authentication实现类）传递给 AuthenticationManager 进行认证。
```java
this.getAuthenticationManager().authenticate(token)
```
4. AuthenticationManager管理一系列的AuthenticationProvider，AuthenticationManager会遍历全部AuthenticationProvider去对Authentication进行认证。
5. AuthenticationProvider会调用userDetailService去数据库中验证用户信息 ,返回一个userDetail对象（可自定义），认证成功后返回一个封装了用户权限信息的，以UsernamePasswordAuthenticationToken实现的带用户名和密码以及权限的Authentication 对象。
```java
//appServer为权限信息
return new UsernamePasswordAuthenticationToken(userDetail,null,appServer;
```
6. 已经进行认证的Authentication返回到UsernamePasswordAuthenticationFilter中，如果验证失败，则进入unsuccessfulAuthentication;如果验证成功，则进入successfulAuthentication进行生成token.

# Filter

```java
public class AjaxLoginAuthenticationFilter extends AbstractAuthenticationProcessingFilter {
    private final AuthenticationSuccessHandler successHandler;
    private final AuthenticationFailureHandler failureHandler;

    public AjaxLoginAuthenticationFilter(String defaultProcessUrl, AuthenticationSuccessHandler successHandler,
            AuthenticationFailureHandler failureHandler) {
        this.successHandler = successHandler;
        this.failureHandler = failureHandler;

    }
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
            throws AuthenticationException, IOException, ServletException {
        AjaxLoginRequest loginRequest;
        try{
            if(secretLogin){
                SecretFormRequest secretFormRequest = JSONObject.parseObject(request.getInputStream(),SecretFormRequest.class);
                loginRequest = secretFormRequest.build();
            }else{
                loginRequest = JSONObject.parseObject(request.getInputStream(),AjaxLoginRequest.class);
            }
        }catch (Exception e){
            logger.error(e.getMessage(),e);
            throw new InvalidLoginException(e.getMessage());
        }
        if (StringUtils.isBlank(loginRequest.getUsername()) || StringUtils.isBlank(loginRequest.getPassword())) {
            throw new AuthenticationServiceException("Username or Password not provided");
        }
        //权限标识，信息，用于做权限区分
        AppServer appServer; 

        UsernamePasswordAuthenticationAppServerToken token = new UsernamePasswordAuthenticationAppServerToken(loginRequest.getUsername(), loginRequest.getPassword(),appServer);

        return this.getAuthenticationManager().authenticate(token);
    }

    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain,
            Authentication authResult) throws IOException, ServletException {
        successHandler.onAuthenticationSuccess(request, response, authResult);
    }

    @Override
    protected void unsuccessfulAuthentication(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException failed) throws IOException, ServletException {
        SecurityContextHolder.clearContext();
        failureHandler.onAuthenticationFailure(request, response, failed);
    }

```

# Token（Authentication实现类）
```java
/**
 * 用户认证Token
 */
public class UsernamePasswordAuthenticationAppServerToken extends UsernamePasswordAuthenticationToken {
    //权限标识，信息，用于做权限区分
    private AppServer appServer;

    public UsernamePasswordAuthenticationAppServerToken(Object principal, Object credentials,AppServer appServer){
        this(principal,credentials);
        this.appServer = appServer;
    }

    public UsernamePasswordAuthenticationAppServerToken(Object principal, Object credentials) {
        super(principal, credentials);
    }


    public UsernamePasswordAuthenticationAppServerToken(Object principal, Object credentials, Collection<? extends GrantedAuthority> authorities) {
        super(principal, credentials, authorities);
    }

    public AppServer getAppServer() {
        return appServer;
    }
}
```

# AuthenticationProvider
```java
/**
 * 权限验证对比用户名,密码比较是否相同
 */
@Component
public class AjaxAuthenticationProvider implements AuthenticationProvider {

    @Autowired
    private BCryptPasswordEncoder encoder;
    @Autowired
    private UserDetailsServiceImpl userService;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        Assert.notNull(authentication, "No authentication data provided");

        String username = (String) authentication.getPrincipal();
        String password = (String) authentication.getCredentials();

        JwtUser jwtUser = null;
        AppServer appServer = null;


        jwtUser = (JwtUser) userService.loadUserByUsername(username);

        logger.info("获取用户成功！",jwtUser.getUsername());

        if(authentication instanceof UsernamePasswordAuthenticationAppServerToken){

            UsernamePasswordAuthenticationAppServerToken authenticationAppServerToken = (UsernamePasswordAuthenticationAppServerToken) authentication;
            appServer = authenticationAppServerToken.getAppServer();
            jwtUser = userService.loadUserByUsernameInAppServer(username,appServer.getId(),0L);

        }

        //匹配密码
        if (!encoder.matches(password, jwtUser.getPassword())) {
            throw new BadCredentialsException("Authentication Failed. Username or Password not valid.");
        }

        UsernamePasswordAuthenticationAppServerToken authenticationAppServerToken = new UsernamePasswordAuthenticationAppServerToken(jwtUser,null,appServer);

        return authenticationAppServerToken;
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return (UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication));
    }
}
```

# UserDetailService
```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService, IService<User> {
    private final UserRepository userRepository;
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUserName(username);

        if (user == null) {
            throw new UsernameNotFoundException(String.format("No user found with username '%s'.", username));
        } else {
            return JwtUserFactory.create(user,0L,0L);
        }
    }
}
```
# UserDetail

```java
public class JwtUser implements UserDetails {
    private String username;
    private String password;
    private AppServer appServer;
}

```

# SuccessHandler

```java
@Component
public class AjaxAwareAuthenticationSuccessHandler implements AuthenticationSuccessHandler {

    @Autowired
    private JwtTokenUtil jwtTokenUtil;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,
                                        Authentication authentication) throws IOException, ServletException {
        JwtUser userDetails = (JwtUser) authentication.getPrincipal();
        
        AppServer appServer = null;
        if (authentication instanceof UsernamePasswordAuthenticationAppServerToken) {
            appServer = ((UsernamePasswordAuthenticationAppServerToken) authentication).getAppServer();
        }

        //生成token
        JwtToken accessToken = jwtTokenUtil.generateToken(userDetails,null,null);
            

        userCacheManager.cache(accessToken.getToken(),userDetails);
        if(userDetails.getLastPasswordResetType() != null && userDetails.getLastPasswordResetType()==0 
            		&& userDetails.getLastPasswordResetDate() == null) {
            userService.updateUserResetDate(new Date(), userDetails.getId());  
        }
            //用于是否需要重置密码判断
        accessToken.setLastPasswordResetType(userDetails.getLastPasswordResetType());

        response.setStatus(HttpStatus.OK.value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        JSONObject.writeJSONString(response.getWriter(), accessToken);
    }

```


