---
{"dg-publish":true,"permalink":"/spring/spring-security-03/","dgPassFrontmatter":true}
---

# 登录实践

## csrf未关闭导致的错误

现象：url被重定向到/error

通过一次debug找出错误：
https://www.bilibili.com/video/BV1pg411X7ZP/?p=8&spm_id_from=pageDriver  时间37:02

## 自定义短信验证码登录

### SmsCodeAuthenticationFilter

设置短信验证码登录的过滤器SmsCodeAuthenticationFilter
```Java
public class SmsCodeAuthenticationFilter extends AbstractAuthenticationProcessingFilter {  
    //传一个sms登录的页面  
    protected SmsCodeAuthenticationFilter(String defaultFilterProcessesUrl) {  
        super(defaultFilterProcessesUrl);  
    }  
  
    @Override  
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException, IOException, ServletException {  
  
        String mobilePhone = request.getParameter("mobilePhone");  
        String code = request.getParameter("code");  
        //创建token  
        SmsCodeAuthenticationToken smsCodeAuthenticationToken = new SmsCodeAuthenticationToken(mobilePhone, code);  
        smsCodeAuthenticationToken.setDetails(this.authenticationDetailsSource.buildDetails(request));  
        //进行认证并return结果  
        return this.getAuthenticationManager().authenticate(smsCodeAuthenticationToken);  
    }  
}
```


### SmsCodeAuthenticationToken

仿照UsernamePasswordAuthenticationToken设置我们自己的SmsCodeAuthenticationToken
```Java
public class SmsCodeAuthenticationToken extends AbstractAuthenticationToken {  
  
    private final Object principal;  
    private Object credentials;  
  
    public SmsCodeAuthenticationToken(Object principal, Object credentials) {  
        super(null);  
        this.principal = principal;  
        this.credentials = credentials;  
    }  
    public SmsCodeAuthenticationToken(Object principal, Object credentials,Collection<? extends GrantedAuthority> authorities) {  
        super(authorities);  
        this.principal = principal;  
        this.credentials = credentials;  
        super.setAuthenticated(true);  
    }  
  
    @Override  
    public Object getCredentials() {  
        return this.credentials;  
    }  
  
    @Override  
    public Object getPrincipal() {  
        return this.principal;  
    }  
}
```

### SmsCodeAuthenticationProvider

实现自己的SmsCodeAuthenticationProvider

因为在默认的[[Spring/Spring_Security_01#AbstractUserDetailsAuthenticationProvider\|AbstractUserDetailsAuthenticationProvider]]的[[Spring/Spring_Security_01#🌟authenticate方法\|authenticate]]方法中，
第一行就Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class...)，
说明改provider只对默认的UsernamePasswordAuthenticationToken有效，
因此我们要实现自己的provider

```Java
@Slf4j  
public class SmsCodeAuthenticationProvider implements AuthenticationProvider {  
    @Override  
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {  
        String mobilePhone = (String) authentication.getPrincipal();  
        String code = (String) authentication.getCredentials();  
  
        log.info("手机号【{}】验证码【{}】开始登录",mobilePhone,code);  
        //这里用于比对验证码的正确性，此处直接判断是否等于123456
        if (StringUtils.equals(code,"123456")) {  
            return new SmsCodeAuthenticationToken(mobilePhone,code, Collections.emptyList());  
        }  
  
        //SpringSecurity规范：认证失败就抛异常，不要返回null  
        throw new BadCredentialsException("验证码错误");  
    }  
  
    @Override  
    public boolean supports(Class<?> authentication) {  
        return SmsCodeAuthenticationToken.class.isAssignableFrom(authentication);  
    }  
}
```

### SmsCodeLoginConfigurer

#### 问题

在构建SmsCodeLoginConfigurer时，如果直接调用super构造器传入我们的自定义filter，运行会产生如下错误
```Java
Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.security.web.DefaultSecurityFilterChain]: Factory method 'securityFilterChain' threw exception with message: The Filter class com.xiang.support.SmsCodeAuthenticationFilter does not have a registered order and cannot be added without a specified order. Consider using addFilterBefore or addFilterAfter instead.
	at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:171) ~[spring-beans-6.0.13.jar:6.0.13]
	at org.springframework.beans.factory.support.ConstructorResolver.instantiate(ConstructorResolver.java:650) ~[spring-beans-6.0.13.jar:6.0.13]
	... 19 common frames omitted
Caused by: java.lang.IllegalArgumentException: The Filter class com.xiang.support.SmsCodeAuthenticationFilter does not have a registered order and cannot be added without a specified order. Consider using addFilterBefore or addFilterAfter instead.
	at org.springframework.security.config.annotation.web.builders.HttpSecurity.addFilter(HttpSecurity.java:3298) ~[spring-security-config-6.1.5.jar:6.1.5]
	at org.springframework.security.config.annotation.web.builders.HttpSecurity.addFilter(HttpSecurity.java:143) ~[spring-security-config-6.1.5.jar:6.1.5]
	at org.springframework.security.config.annotation.web.configurers.AbstractAuthenticationFilterConfigurer.configure(AbstractAuthenticationFilterConfigurer.java:302) ~[spring-security-config-6.1.5.jar:6.1.5]
	at org.springframework.security.config.annotation.web.configurers.AbstractAuthenticationFilterConfigurer.configure(AbstractAuthenticationFilterConfigurer.java:61) ~[spring-security-config-6.1.5.jar:6.1.5]
	at org.springframework.security.config.annotation.AbstractConfiguredSecurityBuilder.configure(AbstractConfiguredSecurityBuilder.java:355) ~[spring-security-config-6.1.5.jar:6.1.5]
	at org.springframework.security.config.annotation.AbstractConfiguredSecurityBuilder.doBuild(AbstractConfiguredSecurityBuilder.java:309) ~[spring-security-config-6.1.5.jar:6.1.5]
	at org.springframework.security.config.annotation.AbstractSecurityBuilder.build(AbstractSecurityBuilder.java:38) ~[spring-security-config-6.1.5.jar:6.1.5]
	at com.xiang.config.SecurityConfig.securityFilterChain(SecurityConfig.java:49) ~[classes/:na]
	at com.xiang.config.SecurityConfig$$SpringCGLIB$$0.CGLIB$securityFilterChain$0(<generated>) ~[classes/:na]
	at com.xiang.config.SecurityConfig$$SpringCGLIB$$FastClass$$1.invoke(<generated>) ~[classes/:na]
	at org.springframework.cglib.proxy.MethodProxy.invokeSuper(MethodProxy.java:258) ~[spring-core-6.0.13.jar:6.0.13]
	at org.springframework.context.annotation.ConfigurationClassEnhancer$BeanMethodInterceptor.intercept(ConfigurationClassEnhancer.java:331) ~[spring-context-6.0.13.jar:6.0.13]
	at com.xiang.config.SecurityConfig$$SpringCGLIB$$0.securityFilterChain(<generated>) ~[classes/:na]
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:na]
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77) ~[na:na]
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:na]
	at java.base/java.lang.reflect.Method.invoke(Method.java:568) ~[na:na]
	at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:139) ~[spring-beans-6.0.13.jar:6.0.13]
	... 20 common frames omitted
```

打开[[Spring/Spring_Security_01#HttpSecurity\|HttpSecurity]].java:3298，发现在addFilter方法中，
```Java
@Override  
public HttpSecurity addFilter(Filter filter) {  
    Integer order = this.filterOrders.getOrder(filter.getClass());  
    if (order == null) {  
       throw new IllegalArgumentException("The Filter class " + filter.getClass().getName()  
             + " does not have a registered order and cannot be added without a specified order. Consider using addFilterBefore or addFilterAfter instead.");  
    }  
    this.filters.add(new OrderedFilter(filter, order));  
    return this;  
}
```
order == null，故抛出异常。

#### 原因

过滤器链中的filter是有顺序的

找到getOrder方法所在的类FilterOrderRegistration
```Java
final class FilterOrderRegistration {  
  
    private static final int INITIAL_ORDER = 100;  
	//步长为100
    private static final int ORDER_STEP = 100;  
  
    private final Map<String, Integer> filterToOrder = new HashMap<>();  
  
    FilterOrderRegistration() {  
	    //每执行一次next增加100
       Step order = new Step(INITIAL_ORDER, ORDER_STEP);  
       put(DisableEncodeUrlFilter.class, order.next());  
       put(ForceEagerSessionCreationFilter.class, order.next());  
       put(ChannelProcessingFilter.class, order.next());  
       order.next(); // gh-8105  
       put(WebAsyncManagerIntegrationFilter.class, order.next());  
       put(SecurityContextHolderFilter.class, order.next());  
       put(SecurityContextPersistenceFilter.class, order.next());  
       put(HeaderWriterFilter.class, order.next());  
       put(CorsFilter.class, order.next());  
       put(CsrfFilter.class, order.next());  
       put(LogoutFilter.class, order.next());  
       this.filterToOrder.put(  
             "org.springframework.security.oauth2.client.web.OAuth2AuthorizationRequestRedirectFilter",  
             order.next());  
       this.filterToOrder.put(  
             "org.springframework.security.saml2.provider.service.web.Saml2WebSsoAuthenticationRequestFilter",  
             order.next());  
       put(X509AuthenticationFilter.class, order.next());  
       put(AbstractPreAuthenticatedProcessingFilter.class, order.next());  
       this.filterToOrder.put("org.springframework.security.cas.web.CasAuthenticationFilter", order.next());  
       this.filterToOrder.put("org.springframework.security.oauth2.client.web.OAuth2LoginAuthenticationFilter",  
             order.next());  
       this.filterToOrder.put(  
             "org.springframework.security.saml2.provider.service.web.authentication.Saml2WebSsoAuthenticationFilter",  
             order.next());  
       put(UsernamePasswordAuthenticationFilter.class, order.next());  
       order.next(); // gh-8105  
       put(DefaultLoginPageGeneratingFilter.class, order.next());  
       put(DefaultLogoutPageGeneratingFilter.class, order.next());  
       put(ConcurrentSessionFilter.class, order.next());  
       put(DigestAuthenticationFilter.class, order.next());  
       this.filterToOrder.put(  
             "org.springframework.security.oauth2.server.resource.web.authentication.BearerTokenAuthenticationFilter",  
             order.next());  
       put(BasicAuthenticationFilter.class, order.next());  
       put(RequestCacheAwareFilter.class, order.next());  
       put(SecurityContextHolderAwareRequestFilter.class, order.next());  
       put(JaasApiIntegrationFilter.class, order.next());  
       put(RememberMeAuthenticationFilter.class, order.next());  
       put(AnonymousAuthenticationFilter.class, order.next());  
       this.filterToOrder.put("org.springframework.security.oauth2.client.web.OAuth2AuthorizationCodeGrantFilter",  
             order.next());  
       put(SessionManagementFilter.class, order.next());  
       put(ExceptionTranslationFilter.class, order.next());  
       put(FilterSecurityInterceptor.class, order.next());  
       put(AuthorizationFilter.class, order.next());  
       put(SwitchUserFilter.class, order.next());  
    }  
  
    void put(Class<? extends Filter> filter, int position) {  
       this.filterToOrder.putIfAbsent(filter.getName(), position);  
    }  
  
    Integer getOrder(Class<?> clazz) {  
       while (clazz != null) {  
          Integer result = this.filterToOrder.get(clazz.getName());  
          if (result != null) {  
             return result;  
          }  
          clazz = clazz.getSuperclass();  
       }  
       return null;  
    }  
  
    private static class Step {  
  
       private int value;  
  
       private final int stepSize;  
  
       Step(int initialValue, int stepSize) {  
          this.value = initialValue;  
          this.stepSize = stepSize;  
       }  
  
       int next() {  
          int value = this.value;  
          this.value += this.stepSize;  
          return value;  
       }  
  
    }  
  
}
```

因此，需要我们为自定义的SmsCodeAuthenticationFilter添加一个顺序，使order不为null

#### 解决

在[[Spring/Spring_Security_01#父类AbstractAuthenticationFilterConfigurer\|AbstractAuthenticationFilterConfigurer]]类中的configure方法中最后调用了addFilter方法，但是SpringSecurity并没有在[[Spring/Spring_Security_01#父类AbstractAuthenticationFilterConfigurer\|AbstractAuthenticationFilterConfigurer]]类中添加addFilter方法，我们无法在自己的SmsCodeLoginConfigurer类中通过addFilter方法加入我们的filter。

>有两种解决方法：

>1.通过反射机制调用HttpSecurity实例的filterOrders属性的put方法，手动添加我们的filter并指定order（推荐）

SmsCodeLoginConfigurer类
```Java
public class SmsCodeLoginConfigurer<H extends HttpSecurityBuilder<H>>  
        extends AbstractAuthenticationFilterConfigurer<H, SmsCodeLoginConfigurer<H>, SmsCodeAuthenticationFilter> {  
  
    public SmsCodeLoginConfigurer(String defaultLoginProcessingUrl) {  
        super(new SmsCodeAuthenticationFilter(defaultLoginProcessingUrl), defaultLoginProcessingUrl);  
    }  
  
    @Override  
    protected RequestMatcher createLoginProcessingUrlMatcher(String loginProcessingUrl) {  
        return new AntPathRequestMatcher(loginProcessingUrl, HttpMethod.POST.name());  
    }  
}
```

config类
```Java
@Configuration  
public class SecurityConfig {  
    @Bean  
    public DefaultSecurityFilterChain securityFilterChain(HttpSecurity http,  
                                                          AuthenticationSuccessHandler authenticationSuccessHandler,  
                                                          AuthenticationFailureHandler authenticationFailureHandler,  
                                                          AuthenticationEntryPoint authenticationEntryPoint) throws Exception {  
        http.formLogin()  
                .usernameParameter("username")  
                .passwordParameter("password")  
                .loginProcessingUrl("/login")  
                .successHandler(authenticationSuccessHandler)  
                .failureHandler(authenticationFailureHandler)  
                .and()  
                .csrf()  
                .disable()  
                //配置权限  
                .authorizeRequests()  
                .anyRequest()  
                .authenticated()  
                .and()  
                //配置登录入口  
                .exceptionHandling()  
                .authenticationEntryPoint(authenticationEntryPoint);  
  
  
        Field filterOrdersField = HttpSecurity.class.getDeclaredField("filterOrders");  
        filterOrdersField.setAccessible(true);  
        Object filterRegistration = filterOrdersField.get(http);  
        Method putMethod = filterRegistration.getClass().getDeclaredMethod("put", Class.class, int.class);  
        putMethod.setAccessible(true);  
        //要把SmsCodeAuthenticationFilter的order放在UsernamePasswordAuthenticationFilter附近  
        //因为UsernamePasswordAuthenticationFilter是1900，所以将其order设为1901  
        putMethod.invoke(filterRegistration, SmsCodeAuthenticationFilter.class,1901);  
  
        SmsCodeLoginConfigurer<HttpSecurity> httpSecuritySmsCodeLoginConfigurer = new SmsCodeLoginConfigurer<>("/smsCodeLogin");  
        httpSecuritySmsCodeLoginConfigurer  
                .loginProcessingUrl("/smsCodeLogin")  
                .successHandler(authenticationSuccessHandler)  
                .failureHandler(authenticationFailureHandler);  
  
        http.apply(httpSecuritySmsCodeLoginConfigurer);  
        http.authenticationProvider(new SmsCodeAuthenticationProvider());  
        return http.build();  
    }   
}
```


>2.重写AbstractAuthenticationFilterConfigurer类的configure方法

SmsCodeLoginConfigurer类
```Java
public class SmsCodeLoginConfigurer2<B extends HttpSecurityBuilder<B>> extends AbstractHttpConfigurer<SmsCodeLoginConfigurer2<B>,B> {  
    private AuthenticationSuccessHandler authenticationSuccessHandler;  
    private AuthenticationFailureHandler authenticationFailureHandler;  
  
    @Override  
    public void configure(B builder) throws Exception {  
        SmsCodeAuthenticationFilter authenticationFilter = new SmsCodeAuthenticationFilter("/smsCodeLogin");  
        //AuthenticationManager在构建的时候会把AuthenticationManager放到SharedObject中  
        authenticationFilter.setAuthenticationManager(builder.getSharedObject(AuthenticationManager.class));  
        authenticationFilter.setAuthenticationSuccessHandler(authenticationSuccessHandler);  
        authenticationFilter.setAuthenticationFailureHandler(authenticationFailureHandler);  
        authenticationFilter.setAuthenticationDetailsSource(new WebAuthenticationDetailsSource());  
  
        builder.addFilterAfter(authenticationFilter, UsernamePasswordAuthenticationFilter.class);  
  
    }  
  
    public final SmsCodeLoginConfigurer2<B> successHandler(AuthenticationSuccessHandler successHandler) {  
        this.authenticationSuccessHandler = successHandler;  
        return this;  
    }  
  
    public final SmsCodeLoginConfigurer2<B> failureHandler(AuthenticationFailureHandler failureHandler) {  
        this.authenticationFailureHandler = failureHandler;  
        return this;  
    }  
}
```

config类
```Java  
//短信验证码登录第二种配置方式  
   
  
@Configuration  
public class SecurityConfig2 {  
    @Bean  
    public DefaultSecurityFilterChain securityFilterChain(HttpSecurity http,  
                                                          AuthenticationSuccessHandler authenticationSuccessHandler,  
                                                          AuthenticationFailureHandler authenticationFailureHandler,  
                                                          AuthenticationEntryPoint authenticationEntryPoint) throws Exception {  
        http.formLogin()  
                .usernameParameter("username")  
                .passwordParameter("password")  
                .loginProcessingUrl("/login")  
                .successHandler(authenticationSuccessHandler)  
                .failureHandler(authenticationFailureHandler)  
                .and()  
                .csrf()  
                .disable()  
                //配置权限  
                .authorizeRequests()  
                .anyRequest()  
                .authenticated()  
                .and()  
                //配置登录入口  
                .exceptionHandling()  
                .authenticationEntryPoint(authenticationEntryPoint);  
  
  
  
        SmsCodeLoginConfigurer2<HttpSecurity> smsCodeLoginConfigurer2 = new SmsCodeLoginConfigurer2<>();  
        smsCodeLoginConfigurer2  
                .successHandler(authenticationSuccessHandler)  
                .failureHandler(authenticationFailureHandler);  
        http.apply(smsCodeLoginConfigurer2);  
  
        http.authenticationProvider(new SmsCodeAuthenticationProvider());  
        return http.build();  
    }  
}
```


# 关于@Configuration注解的proxyBeanMethods属性

@Configuration中有属性proxyBeanMethods，默认为true，即是否使用类中bean的代理bean

在含有@Configuration(proxyBeanMethods = false)的类中，
假设有两个bean，记为xxx和yyy
如果yyy有需要注入xxx的属性，

```Java
@Bean
public Xxx xxx() {
	return Inew Xxx() ;
｝

@Bean
public Yyy yyy() {
	Yyy yyy = new Yyy();
	yyy.setXxx(xxx());
}
```
则我们如代码所示手动调用xxx方法为属性赋值
会使yyy的Xxx属性维护一个Xxx的实例，同时spring容器也会维护一个Xxx的实例
这时会破坏xxx的单例。
而proxyBeanMethods默认为true，使用代理类就可以避免这个问题

