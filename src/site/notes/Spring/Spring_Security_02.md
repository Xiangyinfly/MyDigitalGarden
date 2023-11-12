---
{"dg-publish":true,"permalink":"/spring/spring-security-02/","dgPassFrontmatter":true}
---

# SecurityContext相关类

- ﻿﻿SecurityContext
	存有Authentication的一个对象
- ﻿﻿SecurityContextHolder
- SecurityContextHolderStrategy
- ﻿﻿SecurityContextPersistenceFilter
- ﻿﻿SecurityContextHolderFilter
- ﻿﻿SecurityContextRepository
- ﻿﻿SecurityContextConfigurer


## SecurityContext

存储`Authentication`的实例就是安全上下文

```Java
public interface SecurityContext extends Serializable {  
    Authentication getAuthentication();  
    void setAuthentication(Authentication authentication);  
}
```

## SecurityContextHolder

**SecurityContext的持有器**

### 一些设置context的重要方法

```
public static void clearContext() {  
    strategy.clearContext();  
}

public static SecurityContext getContext() {  
    return strategy.getContext();  
}

public static void setContext(SecurityContext context) {  
    strategy.setContext(context);  
}
```

通过strategy属性来实现这些方法
SecurityContextHolder类维护了一个[[Spring/Spring_Security_02#SecurityContextHolderStrategy\|SecurityContextHolderStrategy]]
```
private static SecurityContextHolderStrategy strategy;
```


### 初始化

类加载后首先执行静态代码块

```Java
static {  
    initialize();  //执行初始化
}  
  
private static void initialize() {  
    initializeStrategy();  
    initializeCount++;  
}  
  
private static void initializeStrategy() {  
	//判断策略类是否在初始化之前已经自定义设置好。返回true则直接退出方法
    if (MODE_PRE_INITIALIZED.equals(strategyName)) {  
       Assert.state(strategy != null, "When using " + MODE_PRE_INITIALIZED  
             + ", setContextHolderStrategy must be called with the fully constructed strategy");  
       return;  
    }  
    //判断是否空
    if (!StringUtils.hasText(strategyName)) {  
       // Set default  
       strategyName = MODE_THREADLOCAL;  //默认为MODE_THREADLOCAL
    }  
    if (strategyName.equals(MODE_THREADLOCAL)) {  
    //将strategy默认设置为ThreadLocalSecurityContextHolderStrategy
       strategy = new ThreadLocalSecurityContextHolderStrategy();  
       return;  
    }  
    if (strategyName.equals(MODE_INHERITABLETHREADLOCAL)) {  
       strategy = new InheritableThreadLocalSecurityContextHolderStrategy();  
       return;  
    }  
    if (strategyName.equals(MODE_GLOBAL)) {  
       strategy = new GlobalSecurityContextHolderStrategy();  
       return;  
    }  
    // Try to load a custom strategy  
    try {  
       Class<?> clazz = Class.forName(strategyName);  
       Constructor<?> customStrategy = clazz.getConstructor();  
       strategy = (SecurityContextHolderStrategy) customStrategy.newInstance();  
    }  
    catch (Exception ex) {  
       ReflectionUtils.handleReflectionException(ex);  
    }  
}
```

## SecurityContextHolderStrategy

**SecurityContextHolder存储SecurityContext的一种策略**

```Java
public interface SecurityContextHolderStrategy {  
    void clearContext();  
  
    SecurityContext getContext();  
  
    default Supplier<SecurityContext> getDeferredContext() {  
       return () -> getContext();  
    }  
    
    void setContext(SecurityContext context);  
  
    default void setDeferredContext(Supplier<SecurityContext> deferredContext) {  
       setContext(deferredContext.get());  
    }  
  
    SecurityContext createEmptyContext();  
  
}
```

实现类：![Pasted image 20231110203006.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231110203006.png)

### 主要使用ThreadLocalSecurityContextHolderStrategy

一个变量对一个线程共享，对不同线程不共享

```Java
final class ThreadLocalSecurityContextHolderStrategy implements SecurityContextHolderStrategy {  
  
    private static final ThreadLocal<Supplier<SecurityContext>> contextHolder = new ThreadLocal<>();  
  
    @Override  
    public void clearContext() {  
       contextHolder.remove();  
    }  
  
    @Override  
    public SecurityContext getContext() {  
       return getDeferredContext().get();  
    }  
  
    @Override  
    public Supplier<SecurityContext> getDeferredContext() {  
       Supplier<SecurityContext> result = contextHolder.get();  
       if (result == null) {  
          SecurityContext context = createEmptyContext();  
          result = () -> context;  
          contextHolder.set(result);  
       }  
       return result;  
    }  
  
    @Override  
    public void setContext(SecurityContext context) {  
       Assert.notNull(context, "Only non-null SecurityContext instances are permitted");  
       contextHolder.set(() -> context);  
    }  
  
    @Override  
    public void setDeferredContext(Supplier<SecurityContext> deferredContext) {  
       Assert.notNull(deferredContext, "Only non-null Supplier instances are permitted");  
       Supplier<SecurityContext> notNullDeferredContext = () -> {  
          SecurityContext result = deferredContext.get();  
          Assert.notNull(result, "A Supplier<SecurityContext> returned null and is not allowed.");  
          return result;  
       };  
       contextHolder.set(notNullDeferredContext);  
    }  
  
    @Override  
    public SecurityContext createEmptyContext() {  
       return new SecurityContextImpl();  //是SecurityContext的默认实现
    }  
  
}
```


## SecurityContextRepository

用于持久化SecurityContext

常常自定义SecurityContextRepository，通过内存数据库来进行持久化

```Java
public interface SecurityContextRepository {  
  
	@Deprecated  
    SecurityContext loadContext(HttpRequestResponseHolder requestResponseHolder);  
	//获取
    default DeferredSecurityContext loadDeferredContext(HttpServletRequest request) {  
       Supplier<SecurityContext> supplier = () -> loadContext(new HttpRequestResponseHolder(request, null));  
       return new SupplierDeferredSecurityContext(SingletonSupplier.of(supplier),  
             SecurityContextHolder.getContextHolderStrategy());  
    }   
    //保存
    void saveContext(SecurityContext context, HttpServletRequest request, HttpServletResponse response);  
  
    boolean containsContext(HttpServletRequest request);  
  
}```

## SecurityContextPersistenceFilter

该类已经过时，用[[Spring/Spring_Security_02#SecurityContextHolderFilter\|SecurityContextHolderFilter]]代替

![Pasted image 20231110213000.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231110213000.png)

### doFilter方法
```Java
@Override  
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)  
       throws IOException, ServletException {  
    doFilter((HttpServletRequest) request, (HttpServletResponse) response, chain);  
}  
  
private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)  
       throws IOException, ServletException {  
    // ensure that filter is only applied once per request  
    //保证每个请求只会进行一次filter
    if (request.getAttribute(FILTER_APPLIED) != null) {  
       chain.doFilter(request, response);  
       return;  
    }  
    request.setAttribute(FILTER_APPLIED, Boolean.TRUE);  
    if (this.forceEagerSessionCreation) {  
       HttpSession session = request.getSession();  
       if (this.logger.isDebugEnabled() && session.isNew()) {  
          this.logger.debug(LogMessage.format("Created session %s eagerly", session.getId()));  
       }  
    }  
    HttpRequestResponseHolder holder = new HttpRequestResponseHolder(request, response);  
    SecurityContext contextBeforeChainExecution = this.repo.loadContext(holder);  
    try {  
	    //调用ThreadLocalSecurityContextHolderStrategy的setContext方法。
		//方便后续可以直接可以用securityContextHolderStrategy获取SecurityContext
       this.securityContextHolderStrategy.setContext(contextBeforeChainExecution);  
       if (contextBeforeChainExecution.getAuthentication() == null) {  
          logger.debug("Set SecurityContextHolder to empty SecurityContext");  
       }  
       else {  
          if (this.logger.isDebugEnabled()) {  
             this.logger  
                .debug(LogMessage.format("Set SecurityContextHolder to %s", contextBeforeChainExecution));  
          }  
       }  
       chain.doFilter(holder.getRequest(), holder.getResponse());  
    }  
    finally {  
       SecurityContext contextAfterChainExecution = this.securityContextHolderStrategy.getContext();  
       // Crucial removal of SecurityContextHolder contents before anything else.  
       //最后清除Context
       this.securityContextHolderStrategy.clearContext();  
       this.repo.saveContext(contextAfterChainExecution, holder.getRequest(), holder.getResponse());  
       request.removeAttribute(FILTER_APPLIED);  
       this.logger.debug("Cleared SecurityContextHolder to complete request");  
    }  
}
```

## SecurityContextHolderFilter

```Java
@Override  
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)  
       throws IOException, ServletException {  
    doFilter((HttpServletRequest) request, (HttpServletResponse) response, chain);  
}  
  
private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)  
       throws ServletException, IOException {  
    if (request.getAttribute(FILTER_APPLIED) != null) {  
       chain.doFilter(request, response);  
       return;  
    }  
    request.setAttribute(FILTER_APPLIED, Boolean.TRUE);  
    Supplier<SecurityContext> deferredContext = this.securityContextRepository.loadDeferredContext(request);  
    try {  
       this.securityContextHolderStrategy.setDeferredContext(deferredContext);  
       chain.doFilter(request, response);  
    }  
    finally {  
       this.securityContextHolderStrategy.clearContext();  
       request.removeAttribute(FILTER_APPLIED);  
    }  
}
```

doFilter方法中不会去保存SecurityContext
需要我们自己保存，比如在AuthenticationSuccessHandler中去保存

## SecurityContextConfigurer

![Pasted image 20231110223321.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231110223321.png)

配置过滤器

### configure方法
```Java
@Override  
@SuppressWarnings("unchecked")  
public void configure(H http) {  
    SecurityContextRepository securityContextRepository = getSecurityContextRepository();  
    if (this.requireExplicitSave) { //判断是否需要手动保存，默认false
	    //返回true，即需要手动保存 
	    //可以自定义ObjectPostProcess
       SecurityContextHolderFilter securityContextHolderFilter = postProcess(  
             new SecurityContextHolderFilter(securityContextRepository));  
       securityContextHolderFilter.setSecurityContextHolderStrategy(getSecurityContextHolderStrategy());  
       http.addFilter(securityContextHolderFilter);  
    }  
    else {  
       SecurityContextPersistenceFilter securityContextFilter = new SecurityContextPersistenceFilter(  
             securityContextRepository);  
       securityContextFilter.setSecurityContextHolderStrategy(getSecurityContextHolderStrategy());  
       SessionManagementConfigurer<?> sessionManagement = http.getConfigurer(SessionManagementConfigurer.class);  
       SessionCreationPolicy sessionCreationPolicy = (sessionManagement != null)  
             ? sessionManagement.getSessionCreationPolicy() : null;  
       if (SessionCreationPolicy.ALWAYS == sessionCreationPolicy) {  
          securityContextFilter.setForceEagerSessionCreation(true);  
          http.addFilter(postProcess(new ForceEagerSessionCreationFilter()));  
       }  
       securityContextFilter = postProcess(securityContextFilter);  
       http.addFilter(securityContextFilter);  
    }  
}
```

**build方法 -> doBuild方法 -> performBuild方法 -> 执行真正的逻辑**

## SecurityFilterChain加入FilterChainProxy的时机

SpringSecurity顶层为FilterChainProxy，维护了一堆SecurityFilterChain
HttpSecurity的performBuild方法构建出了SecurityFilterChain的默认实现DefaultSecurityFilterChain
WebSecurity的performBuild方法构建出了一个Filter，本质上是一个FilterChainProxy

>什么时候将一堆SecurityFilterChain加入该FilterChainProxy？

WebSecurity的performBuild方法
```Java
@Override  
protected Filter performBuild() throws Exception {  
    Assert.state(!this.securityFilterChainBuilders.isEmpty(),  
          () -> "At least one SecurityBuilder<? extends SecurityFilterChain> needs to be specified. "  
                + "Typically this is done by exposing a SecurityFilterChain bean. "  
                + "More advanced users can invoke " + WebSecurity.class.getSimpleName()  
                + ".addSecurityFilterChainBuilder directly");  
    int chainSize = this.ignoredRequests.size() + this.securityFilterChainBuilders.size();  
    //创建SecurityFilterChain的集合
    List<SecurityFilterChain> securityFilterChains = new ArrayList<>(chainSize);  
    List<RequestMatcherEntry<List<WebInvocationPrivilegeEvaluator>>> requestMatcherPrivilegeEvaluatorsEntries = new ArrayList<>();  
    for (RequestMatcher ignoredRequest : this.ignoredRequests) {  
       WebSecurity.this.logger.warn("You are asking Spring Security to ignore " + ignoredRequest  
             + ". This is not recommended -- please use permitAll via HttpSecurity#authorizeHttpRequests instead."); 
	    //构建securityFilterChain并加入
       SecurityFilterChain securityFilterChain = new DefaultSecurityFilterChain(2️⃣ignoredRequest);  
       securityFilterChains.add(securityFilterChain);  
       requestMatcherPrivilegeEvaluatorsEntries  
          .add(getRequestMatcherPrivilegeEvaluatorsEntry(securityFilterChain));  
    }  
    //3️⃣
    for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : this.securityFilterChainBuilders) {  
       SecurityFilterChain securityFilterChain = securityFilterChainBuilder.build();  
       securityFilterChains.add(securityFilterChain);  
       requestMatcherPrivilegeEvaluatorsEntries  
          .add(getRequestMatcherPrivilegeEvaluatorsEntry(securityFilterChain));  
    }  
    if (this.privilegeEvaluator == null) {  
       this.privilegeEvaluator = new RequestMatcherDelegatingWebInvocationPrivilegeEvaluator(  
             requestMatcherPrivilegeEvaluatorsEntries);  
    }  
    //将securityFilterChains加入到filterChainProxy
    FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);  
    if (this.httpFirewall != null) {  
       filterChainProxy.setFirewall(this.httpFirewall);  
    }  
    if (this.requestRejectedHandler != null) {  
       filterChainProxy.setRequestRejectedHandler(this.requestRejectedHandler);  
    }  
    else if (!this.observationRegistry.isNoop()) {  
       CompositeRequestRejectedHandler requestRejectedHandler = new CompositeRequestRejectedHandler(  
             new ObservationMarkingRequestRejectedHandler(this.observationRegistry),  
             new HttpStatusRequestRejectedHandler());  
       filterChainProxy.setRequestRejectedHandler(requestRejectedHandler);  
    }  
    filterChainProxy.setFilterChainDecorator(getFilterChainDecorator());  
    filterChainProxy.afterPropertiesSet();  
  
    Filter result = filterChainProxy;  
    if (this.debugEnabled) {  
       this.logger.warn("\n\n" + "********************************************************************\n"  
             + "**********        Security debugging is enabled.       *************\n"  
             + "**********    This may include sensitive information.  *************\n"  
             + "**********      Do not use in a production system!     *************\n"  
             + "********************************************************************\n\n");  
       result = new DebugFilter(filterChainProxy);  
    }  
  
    this.postBuildAction.run();  
    return result;  
}
```

2️⃣关于ignoredRequest
WebSecurity自己维护的属性，本质上是RequestMatcher

在我们所写的config类中，
```
@Bean  
public WebSecurityCustomizer webSecurityCustomizer() {  
    return web -> {  
	    web.ignoring().mvcMatchers();
    }  
}
```
配置忽略的请求保存在ignoredRequest中

3️⃣
我们调用addSecurityFilterChainBuilder方法可以在securityFilterChainBuilders属性中加入SecurityFilterChainBuilder，其中的SecurityFilterChainBuilder会在此自动构建产生SecurityFilterChain


## SpringSecurity本质是一个过滤器之中套了很多过滤器

FilterChainProxy的doFilter方法
	FilterChainProxy的doFilterInternal方法
		FilterChainProxy的getFilter方法
			DefaultSecurityFilterChain的matches方法
			this.requestMatcher匹配anyRequest，一定返回true
		构建VirtualFilterChain，逐个filter进行doFilter
		VirtualFilterChain中的filter逐个doFilter结束后，执行this.originalChain.doFilter
		

VirtualFilterChain中的doFilter方法：
```Java
@Override  
public void doFilter(ServletRequest request, ServletResponse response) throws IOException, ServletException {  
    if (this.currentPosition == this.size) {  
       this.originalChain.doFilter(request, response);  
       return;  
    }  
    this.currentPosition++;  
    Filter nextFilter = this.additionalFilters.get(this.currentPosition - 1);  
    if (logger.isTraceEnabled()) {  
       String name = nextFilter.getClass().getSimpleName();  
       logger.trace(LogMessage.format("Invoking %s (%d/%d)", name, this.currentPosition, this.size));  
    }  
    nextFilter.doFilter(request, response, this);  
}
```


---

# RememberMe和Session相关类

- ﻿﻿RememberMeAuthenticationFilter
- ﻿﻿RememberMeAuthenticationProvider
- ﻿﻿RememberMeAuthenticationToken
- ﻿﻿RememberMeConfigurer
- ﻿﻿RememberMeServices
- ﻿﻿SessionManagementConfigurer
- ﻿﻿SessionManagementFilter
- ﻿SessionAuthenticationStrategy
- ﻿﻿SessionInformation
- ﻿﻿SessionRegistry
- ﻿﻿SessionInformationExpiredStrategy

RememberMe也算一种认证方式，需要cookie认证

## RememberMeAuthenticationFilter

### 属性
```Java
private SecurityContextHolderStrategy securityContextHolderStrategy = SecurityContextHolder  
    .getContextHolderStrategy();  
  
private ApplicationEventPublisher eventPublisher;  
  
private AuthenticationSuccessHandler successHandler;  
  
private AuthenticationManager authenticationManager;  
  
private RememberMeServices rememberMeServices;  
  
private SecurityContextRepository securityContextRepository = new HttpSessionSecurityContextRepository();
```


### doFilter方法

```Java
@Override  
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)  
       throws IOException, ServletException {  
    doFilter((HttpServletRequest) request, (HttpServletResponse) response, chain);  
}  
  
private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)  
       throws IOException, ServletException {  
    if (this.securityContextHolderStrategy.getContext().getAuthentication() != null) {  //如果不等于null，说明已经被认证过，就不需要认证了，退出方法
       this.logger.debug(LogMessage  
          .of(() -> "SecurityContextHolder not populated with remember-me token, as it already contained: '"  
                + this.securityContextHolderStrategy.getContext().getAuthentication() + "'"));  
       chain.doFilter(request, response);  
       return;  
    }  
    //1️⃣
    Authentication rememberMeAuth = this.rememberMeServices.autoLogin(request, response);  
    if (rememberMeAuth != null) {  
       // Attempt authentication via AuthenticationManager  
       try {  
	       //调用authenticate方法，得到一个认证后的Authentication重新赋值给rememberMeAuth
          rememberMeAuth = this.authenticationManager.authenticate(rememberMeAuth);  
          // Store to SecurityContextHolder  

		  //创建一个新的context并放入rememberMeAuth，再setContext
          SecurityContext context = this.securityContextHolderStrategy.createEmptyContext();  
          context.setAuthentication(rememberMeAuth);  
          this.securityContextHolderStrategy.setContext(context);  
          onSuccessfulAuthentication(request, response, rememberMeAuth);  
          this.logger.debug(LogMessage.of(() -> "SecurityContextHolder populated with remember-me token: '"  
                + this.securityContextHolderStrategy.getContext().getAuthentication() + "'"));
          //进行持久化
          this.securityContextRepository.saveContext(context, request, response);  
          if (this.eventPublisher != null) {  
             this.eventPublisher.publishEvent(new InteractiveAuthenticationSuccessEvent(  
                   this.securityContextHolderStrategy.getContext().getAuthentication(), this.getClass()));  
          }  
          if (this.successHandler != null) {  
             this.successHandler.onAuthenticationSuccess(request, response, rememberMeAuth);  
             return;  
          }  
       }  
       catch (AuthenticationException ex) {  
          this.logger.debug(LogMessage  
             .format("SecurityContextHolder not populated with remember-me token, as AuthenticationManager "  
                   + "rejected Authentication returned by RememberMeServices: '%s'; "  
                   + "invalidating remember-me token", rememberMeAuth),  
                ex);  
          this.rememberMeServices.loginFail(request, response); 
          //空实现，意味着登录失败不会抛出异常 
          onUnsuccessfulAuthentication(request, response, ex);  
       }  
    }  
    chain.doFilter(request, response);  
}
```

#### 1️⃣
this.rememberMeServices：
[[Spring/Spring_Security_02#RememberMeServices\|RememberMeServices]] 类属性

autoLogin方法：
[[Spring/Spring_Security_02#autoLogin方法\|autoLogin]]方法的重要实现类[[Spring/Spring_Security_02#AbstractRememberMeServices\|AbstractRememberMeServices]]
## RememberMeServices

### 作用
通过request拿出信息，构造出一个Authentication，和UsernamePasswordAuthenticationToken类似

根据我们的需要构造一个用于rememberMe认证的Authentication

```Java
public interface RememberMeServices {  
	//根据请求信息构造Authentication
    Authentication autoLogin(HttpServletRequest request, HttpServletResponse response);  
  
    void loginFail(HttpServletRequest request, HttpServletResponse response);  
  
    void loginSuccess(HttpServletRequest request, HttpServletResponse response,  
          Authentication successfulAuthentication);  
  
}
```


>The returned Authentication must be acceptable to org.springframework.security.authentication.AuthenticationManager or org.springframework.security.authentication.AuthenticationProvider defined by the web application. It is recommended org.springframework.security.authentication.RememberMeAuthenticationToken be used in most cases, as it has a corresponding authentication provider

建议返回一个[[Spring/Spring_Security_02#RememberMeAuthenticationToken\|RememberMeAuthenticationToken]]

### 实现类
![Pasted image 20231111225059.png|undefined](/img/user/Pasted%20image%2020231111225059.png)

## RememberMeAuthenticationProvider

```Java
@Override  
public Authentication authenticate(Authentication authentication) throws AuthenticationException {  
    if (!supports(authentication.getClass())) {  
       return null;  
    }  
    if (this.key.hashCode() != ((RememberMeAuthenticationToken) authentication).getKeyHash()) {  
       throw new BadCredentialsException(this.messages.getMessage("RememberMeAuthenticationProvider.incorrectKey",  
             "The presented RememberMeAuthenticationToken does not contain the expected key"));  
    }  
    return authentication;  
}
```
该authenticate方法未进行重要逻辑

## RememberMeAuthenticationToken

```Java
public class RememberMeAuthenticationToken extends AbstractAuthenticationToken {  
  
    private static final long serialVersionUID = SpringSecurityCoreVersion.SERIAL_VERSION_UID;  
  
    private final Object principal;  
  
    private final int keyHash;  
  
    public RememberMeAuthenticationToken(String key, Object principal,  
          Collection<? extends GrantedAuthority> authorities) {  
       super(authorities);  
       if ((key == null) || ("".equals(key)) || (principal == null) || "".equals(principal)) {  
          throw new IllegalArgumentException("Cannot pass null or empty values to constructor");  
       }  
       this.keyHash = key.hashCode();  
       this.principal = principal;  
       setAuthenticated(true);  
    }  
  
    private RememberMeAuthenticationToken(Integer keyHash, Object principal,  
          Collection<? extends GrantedAuthority> authorities) {  
       super(authorities);  
       this.keyHash = keyHash;  
       this.principal = principal;  
       setAuthenticated(true);  
    }  
  
    @Override  
    public Object getCredentials() {  
       return "";  
    }  
  
    public int getKeyHash() {  
       return this.keyHash;  
    }  
  
    @Override  
    public Object getPrincipal() {  
       return this.principal;  
    }  
  
    @Override  
    public boolean equals(Object obj) {  
       if (!super.equals(obj)) {  
          return false;  
       }  
       if (obj instanceof RememberMeAuthenticationToken) {  
          RememberMeAuthenticationToken other = (RememberMeAuthenticationToken) obj;  
          if (this.getKeyHash() != other.getKeyHash()) {  
             return false;  
          }  
          return true;  
       }  
       return false;  
    }  
  
    @Override  
    public int hashCode() {  
       int result = super.hashCode();  
       result = 31 * result + this.keyHash;  
       return result;  
    }  
  
}
```

和UsernamePasswordAuthenticationToken一样是AbstractAuthenticationToken的实现类

## AbstractRememberMeServices

### autoLogin方法
```Java
@Override  
public Authentication autoLogin(HttpServletRequest request, HttpServletResponse response) {  
    String rememberMeCookie = extractRememberMeCookie(request);  
    if (rememberMeCookie == null) {  
       return null;  
    }  
    this.logger.debug("Remember-me cookie detected");  
    if (rememberMeCookie.length() == 0) {  
       this.logger.debug("Cookie was empty");  
       cancelCookie(request, response);  
       return null;  
    }  
    try {  
       String[] cookieTokens = decodeCookie(rememberMeCookie);  
       UserDetails user = processAutoLoginCookie(cookieTokens, request, response);  
       this.userDetailsChecker.check(user);  
       this.logger.debug("Remember-me cookie accepted");  
       return createSuccessfulAuthentication(request, user);  
    }  
    catch (CookieTheftException ex) {  
       cancelCookie(request, response);  
       throw ex;  
    }  
    catch (UsernameNotFoundException ex) {  
       this.logger.debug("Remember-me login was valid but corresponding user not found.", ex);  
    }  
    catch (InvalidCookieException ex) {  
       this.logger.debug("Invalid remember-me cookie: " + ex.getMessage());  
    }  
    catch (AccountStatusException ex) {  
       this.logger.debug("Invalid UserDetails: " + ex.getMessage());  
    }  
    catch (RememberMeAuthenticationException ex) {  
       this.logger.debug(ex.getMessage());  
    }  
    cancelCookie(request, response);  
    return null;  
}
```