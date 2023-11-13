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

# RememberMe相关类

- ﻿﻿RememberMeAuthenticationFilter
- ﻿﻿RememberMeAuthenticationProvider
- ﻿﻿RememberMeAuthenticationToken
- ﻿﻿RememberMeConfigurer
- RememberMeServices


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
	//拿到cookie
    String rememberMeCookie = 1️⃣extractRememberMeCookie(request);  
    if (rememberMeCookie == null) {  
       return null;  
    }  
    this.logger.debug("Remember-me cookie detected");  
    if (rememberMeCookie.length() == 0) {  
       this.logger.debug("Cookie was empty");  
       2️⃣cancelCookie(request, response);  
       return null;  
    }  
    try {  
	    //解码
       String[] cookieTokens = 3️⃣decodeCookie(rememberMeCookie);  
       UserDetails user = 4️⃣processAutoLoginCookie(cookieTokens, request, response);  
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

#### 1️⃣extractRememberMeCookie
```Java
protected String extractRememberMeCookie(HttpServletRequest request) {  
    Cookie[] cookies = request.getCookies();  
    if ((cookies == null) || (cookies.length == 0)) {  
       return null;  
    }  
    for (Cookie cookie : cookies) {  
       if (this.cookieName.equals(cookie.getName())) {  
          return cookie.getValue();  
       }  
    }  
    return null;  
}
```

其中this.cookieName为属性：
private String cookieName = SPRING_SECURITY_REMEMBER_ME_COOKIE_KEY;

#### 2️⃣cancelCookie
```Java
protected void cancelCookie(HttpServletRequest request, HttpServletResponse response) {  
    this.logger.debug("Cancelling cookie");  
    Cookie cookie = new Cookie(this.cookieName, null);  
    cookie.setMaxAge(0);  
    cookie.setPath(getCookiePath(request));  
    if (this.cookieDomain != null) {  
       cookie.setDomain(this.cookieDomain);  
    }  
    cookie.setSecure((this.useSecureCookie != null) ? this.useSecureCookie : request.isSecure());  
    response.addCookie(cookie);  
}
```

A positive value indicates that the cookie will expire after that many seconds have passed. Note that the value is the maximum age when the cookie will expire, not the cookie's current age.
A negative value means that the cookie is not stored persistently and will be deleted when the Web browser exits. A zero value causes the cookie to be deleted.

cookie.setMaxAge(0)：设置为0代表删除cookie

#### 3️⃣decodeCookie
```Java
protected String[] decodeCookie(String cookieValue) throws InvalidCookieException { 
	//修复base64编码被抹掉的等号
    for (int j = 0; j < cookieValue.length() % 4; j++) {  
       cookieValue = cookieValue + "=";  
    }  
    String cookieAsPlainText;  
    try {  
       cookieAsPlainText = new String(Base64.getDecoder().decode(cookieValue.getBytes()));  
    }  
    catch (IllegalArgumentException ex) {  
       throw new InvalidCookieException("Cookie token was not Base64 encoded; value was '" + cookieValue + "'");  
    }  
    String[] tokens = StringUtils.delimitedListToStringArray(cookieAsPlainText, DELIMITER); //通过冒号分割 
    for (int i = 0; i < tokens.length; i++) {  
       try {  
          tokens[i] = URLDecoder.decode(tokens[i], StandardCharsets.UTF_8.toString());  
       }  
       catch (UnsupportedEncodingException ex) {  
          this.logger.error(ex.getMessage(), ex);  
       }  
    }  
    return tokens;  
}
```

#### 4️⃣processAutoLoginCookie

在本类中该方法为抽象方法

实现类：
![Pasted image 20231112214932.png|undefined](/img/user/Pasted%20image%2020231112214932.png)
第一个基于数据库，第二个基于内存

在TokenBasedRememberMeServices中的实现：

```Java
@Override  
protected UserDetails processAutoLoginCookie(String[] cookieTokens, HttpServletRequest request,  
       HttpServletResponse response) { 
    //判断cookie长度 
    if (!isValidCookieTokensLength(cookieTokens)) {  
       throw new InvalidCookieException(  
             "Cookie token did not contain 3 or 4 tokens, but contained '" + Arrays.asList(cookieTokens) + "'");  
    }  
    long tokenExpiryTime = getTokenExpiryTime(cookieTokens);  
    if (isTokenExpired(tokenExpiryTime)) {  //判断cookie是否过期
       throw new InvalidCookieException("Cookie token[1] has expired (expired on '" + new Date(tokenExpiryTime)  
             + "'; current time is '" + new Date() + "')");  
    }  
    // Check the user exists. Defer lookup until after expiry time checked, to  
    // possibly avoid expensive database call.    UserDetails userDetails = getUserDetailsService().loadUserByUsername(cookieTokens[0]);  
    Assert.notNull(userDetails, () -> "UserDetailsService " + getUserDetailsService()  
          + " returned null for username " + cookieTokens[0] + ". " + "This is an interface contract violation");  
    // Check signature of token matches remaining details. Must do this after user  
    // lookup, as we need the DAO-derived password. If efficiency was a major issue,    // just add in a UserCache implementation, but recall that this method is usually    // only called once per HttpSession - if the token is valid, it will cause    // SecurityContextHolder population, whilst if invalid, will cause the cookie to    // be cancelled.    String actualTokenSignature = cookieTokens[2];  
    RememberMeTokenAlgorithm actualAlgorithm = this.matchingAlgorithm;  
    // If the cookie value contains the algorithm, we use that algorithm to check the  
    // signature    if (cookieTokens.length == 4) {  
       actualTokenSignature = cookieTokens[3];  
       actualAlgorithm = RememberMeTokenAlgorithm.valueOf(cookieTokens[2]);  
    }  

	//生成一个signature，并于cookie中自带的signature比对
    String expectedTokenSignature = makeTokenSignature(tokenExpiryTime, userDetails.getUsername(), userDetails.getPassword(), actualAlgorithm);  
    if (!equals(expectedTokenSignature, actualTokenSignature)) {  
       throw new InvalidCookieException("Cookie contained signature '" + actualTokenSignature + "' but expected '"  
             + expectedTokenSignature + "'");  
    }  
    return userDetails;  
}
```


## RememberMeConfigurer

重要方法
```Java
@Override  
public void init(H http) throws Exception {  
    validateInput();  
    String key = getKey();  
    RememberMeServices rememberMeServices = getRememberMeServices(http, key);  
    http.setSharedObject(RememberMeServices.class, rememberMeServices);  
    LogoutConfigurer<H> logoutConfigurer = http.getConfigurer(LogoutConfigurer.class);  
    if (logoutConfigurer != null && this.logoutHandler != null) {  
       logoutConfigurer.addLogoutHandler(this.logoutHandler);  
    }  
    RememberMeAuthenticationProvider authenticationProvider = new RememberMeAuthenticationProvider(key);  
    authenticationProvider = postProcess(authenticationProvider);  
    http.authenticationProvider(authenticationProvider);  
    initDefaultLoginFilter(http);  
}  
  
@Override  
public void configure(H http) {  //创建过滤器，加入到过滤器链
    RememberMeAuthenticationFilter rememberMeFilter = new RememberMeAuthenticationFilter(  
          http.getSharedObject(AuthenticationManager.class), this.rememberMeServices);  
    if (this.authenticationSuccessHandler != null) {  
       rememberMeFilter.setAuthenticationSuccessHandler(this.authenticationSuccessHandler);  
    }  
    SecurityContextConfigurer<?> securityContextConfigurer = http.getConfigurer(SecurityContextConfigurer.class);  
    if (securityContextConfigurer != null && securityContextConfigurer.isRequireExplicitSave()) {  
       SecurityContextRepository securityContextRepository = securityContextConfigurer  
          .getSecurityContextRepository();  
       rememberMeFilter.setSecurityContextRepository(securityContextRepository);  
    }  
    rememberMeFilter.setSecurityContextHolderStrategy(getSecurityContextHolderStrategy());  
    rememberMeFilter = postProcess(rememberMeFilter);  
    http.addFilter(rememberMeFilter);  
}
```

剩下的其他方法大多数是给使用者提供自定义属性的方法
即config类中所写的http.xxx()


# Session相关类

- ﻿﻿SessionManagementConfigurer
- ﻿﻿SessionManagementFilter
- ﻿SessionAuthenticationStrategy
- ﻿﻿SessionInformation
- ﻿﻿SessionRegistry
- ﻿﻿SessionInformationExpiredStrategy

## SessionManagementFilter

```Java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)  
       throws IOException, ServletException {  
    doFilter((HttpServletRequest) request, (HttpServletResponse) response, chain);  
}  
  
private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)  
       throws IOException, ServletException {  
    if (request.getAttribute(FILTER_APPLIED) != null) {  
       chain.doFilter(request, response);  
       return;  
    }  
    request.setAttribute(FILTER_APPLIED, Boolean.TRUE);  
    if (!this.securityContextRepository.containsContext(request)) {  
       Authentication authentication = this.securityContextHolderStrategy.getContext().getAuthentication();  
       if (authentication != null && !this.trustResolver.isAnonymous(authentication)) {  
          try {  
             1️⃣this.sessionAuthenticationStrategy.onAuthentication(authentication, request, response);  
          }  
          catch (SessionAuthenticationException ex) {  //抛出异常，失败处理
             // The session strategy can reject the authentication  
             this.logger.debug("SessionAuthenticationStrategy rejected the authentication object", ex);  
             this.securityContextHolderStrategy.clearContext();  
             this.failureHandler.onAuthenticationFailure(request, response, ex);  
             return;  
	      } 
		  this.securityContextRepository.saveContext 
		  (this.securityContextHolderStrategy.getContext(),request, response);  //保存
       }  
       else {  
          if (request.getRequestedSessionId() != null && !request.isRequestedSessionIdValid()) {  
             if (this.logger.isDebugEnabled()) {  
                this.logger.debug(LogMessage.format("Request requested invalid session id %s",  
                      request.getRequestedSessionId()));  
             }  
             if (this.invalidSessionStrategy != null) {  
                this.invalidSessionStrategy.onInvalidSessionDetected(request, response);  
                return;  
             }  
          }  
       }  
    }  
    chain.doFilter(request, response);  
}
```


重要方法：1️⃣this.[[Spring/Spring_Security_02#SessionAuthenticationStrategy\|sessionAuthenticationStrategy]] .onAuthentication

## SessionAuthenticationStrategy

一个策略类的接口
```Java
public interface SessionAuthenticationStrategy {  
  
    void onAuthentication(Authentication authentication, HttpServletRequest request, HttpServletResponse response)  
          throws SessionAuthenticationException;  
  
}
```


实现类有：![Pasted image 20231112222729.png|undefined](/img/user/Pasted%20image%2020231112222729.png)

### 实现类RegisterSessionAuthenticationStrategy

Strategy used to register a user with the SessionRegistry after successful Authentication.

```Java
public class RegisterSessionAuthenticationStrategy implements SessionAuthenticationStrategy {  
  
    private final SessionRegistry sessionRegistry;  
  
    public RegisterSessionAuthenticationStrategy(SessionRegistry sessionRegistry) {  
       Assert.notNull(sessionRegistry, "The sessionRegistry cannot be null");  
       this.sessionRegistry = sessionRegistry;  
    }  
  
     @Override  
    public void onAuthentication(Authentication authentication, HttpServletRequest request,  
          HttpServletResponse response) {  
       //注册一个新的session   
       this.sessionRegistry.registerNewSession(request.getSession().getId(), authentication.getPrincipal());  
    }  
}
```

### 实现类ConcurrentSessionControlAuthenticationStrategy

onAuthentication方法
```Java
@Override  
public void onAuthentication(Authentication authentication, HttpServletRequest request,  
       HttpServletResponse response) {  
    int allowedSessions = getMaximumSessionsForThisUser(authentication);  
    if (allowedSessions == -1) {  
       // We permit unlimited logins  
       return;  
    }  
    //根据principle信息拿到所有的session，bool属性为是否包括过期session
    List<SessionInformation> sessions = this.sessionRegistry.getAllSessions(authentication.getPrincipal(), false);  
    int sessionCount = sessions.size();  
    if (sessionCount < allowedSessions) {  
       // They haven't got too many login sessions running at present  
       return;  
    }  
    if (sessionCount == allowedSessions) {  
       HttpSession session = request.getSession(false);  //false表示如果session不存在不会去创建
       if (session != null) {  
          // Only permit it though if this request is associated with one of the  
          // already registered sessions          
          for (SessionInformation si : sessions) {  
             if (si.getSessionId().equals(session.getId())) {  
                return;  
             }  
          } 
        //如果sessions中没有改session，则说明改session可能是新的session，但是由于sessionCount已经到达允许的最大值了，再加入新的会超出最大值。于是调用allowableSessionsExceeded方法
       }  
       // If the session is null, a new one will be created by the parent class,  
       // exceeding the allowed number    
    }  
    allowableSessionsExceeded(sessions, allowedSessions, this.sessionRegistry);  
}
```

allowableSessionsExceeded方法
```Java
protected void allowableSessionsExceeded(List<SessionInformation> sessions, int allowableSessions,  
       SessionRegistry registry) throws SessionAuthenticationException {  
    if (this.exceptionIfMaximumExceeded || (sessions == null)) {  
       throw new SessionAuthenticationException(  
             this.messages.getMessage("ConcurrentSessionControlAuthenticationStrategy.exceededAllowed",  
                   new Object[] { allowableSessions }, "Maximum sessions of {0} for this principal exceeded"));  
    }  
    // Determine least recently used sessions, and mark them for invalidation  
    sessions.sort(Comparator.comparing(SessionInformation::getLastRequest));  
    int maximumSessionsExceededBy = sessions.size() - allowableSessions + 1;  
    List<SessionInformation> sessionsToBeExpired = sessions.subList(0, maximumSessionsExceededBy);  
    for (SessionInformation session : sessionsToBeExpired) {  
       session.expireNow();  
    }  
}
```

如果不抛异常，即不进if语句，则会把session按照时间排序，将最老的的session标记为过期，然后加入新的session

### 实现类CompositeSessionAuthenticationStrategy

```Java
public class CompositeSessionAuthenticationStrategy implements SessionAuthenticationStrategy {  
  
    private final Log logger = LogFactory.getLog(getClass());  
  
    private final List<SessionAuthenticationStrategy> delegateStrategies;  
  
    public CompositeSessionAuthenticationStrategy(List<SessionAuthenticationStrategy> delegateStrategies) {  
       Assert.notEmpty(delegateStrategies, "delegateStrategies cannot be null or empty");  
       for (SessionAuthenticationStrategy strategy : delegateStrategies) {  
          Assert.notNull(strategy, () -> "delegateStrategies cannot contain null entires. Got " + delegateStrategies);  
       }  
       this.delegateStrategies = delegateStrategies;  
    }  
  
    @Override  
    public void onAuthentication(Authentication authentication, HttpServletRequest request,  
          HttpServletResponse response) throws SessionAuthenticationException {  
       int currentPosition = 0;  
       int size = this.delegateStrategies.size();  
       for (SessionAuthenticationStrategy delegate : this.delegateStrategies) {  
          if (this.logger.isTraceEnabled()) {  
             this.logger.trace(LogMessage.format("Preparing session with %s (%d/%d)",  
                   delegate.getClass().getSimpleName(), ++currentPosition, size));  
          }  
          delegate.onAuthentication(authentication, request, response);  
       }  
    }  
  
    @Override  
    public String toString() {  
       return getClass().getName() + " [delegateStrategies = " + this.delegateStrategies + "]";  
    }  
  
}
```

有属性List<\SessionAuthenticationStrategy> delegateStrategies
维护了一堆的SessionAuthenticationStrategy
再调用该类的onAuthentication方法时会逐个调用delegateStrategies中元素的onAuthentication方法
从而实现多策略的组合模式

### 实现类AbstractSessionFixationProtectionStrategy

```Java
public void onAuthentication(Authentication authentication, HttpServletRequest request,  
       HttpServletResponse response) {  
    boolean hadSessionAlready = request.getSession(false) != null;  
    if (!hadSessionAlready && !this.alwaysCreateSession) {  
       // Session fixation isn't a problem if there's no session  
       return;  
    }  
    // Create new session if necessary  
    HttpSession session = request.getSession();  
    if (hadSessionAlready && request.isRequestedSessionIdValid()) {  
       String originalSessionId;  
       String newSessionId;  
       Object mutex = WebUtils.getSessionMutex(session);  
       synchronized (mutex) {  
          // We need to migrate to a new session  
          originalSessionId = session.getId();  
          //得到一个新的sessionId
          session = applySessionFixation(request);  
          newSessionId = session.getId();  
       }  
       if (originalSessionId.equals(newSessionId)) {  
          this.logger.warn("Your servlet container did not change the session ID when a new session "  
                + "was created. You will not be adequately protected against session-fixation attacks");  
       }  
       else {  
          if (this.logger.isDebugEnabled()) {  
             this.logger.debug(LogMessage.format("Changed session id from %s", originalSessionId));  
          }  
       }  
       onSessionChange(originalSessionId, session, authentication);  
    }  
}
```

通过定期改变sessionId来防止固定会话攻击

#### applySessionFixation方法

```
abstract HttpSession applySessionFixation(HttpServletRequest request);
```
![Pasted image 20231113111435.png|undefined](/img/user/Pasted%20image%2020231113111435.png)

模版方法，有两个实现

##### ChangeSessionIdAuthenticationStrategy中的实现
```Java
public final class ChangeSessionIdAuthenticationStrategy extends AbstractSessionFixationProtectionStrategy {  
  
    @Override  
    HttpSession applySessionFixation(HttpServletRequest request) {  
       request.changeSessionId();  
       return request.getSession();  
    }  
  
}
```
直接改变sessionId，不用把session销毁也不会创建新的session对象，即session对象不会改变

##### SessionFixationProtectionStrategy中的实现
```Java
@Override  
final HttpSession applySessionFixation(HttpServletRequest request) {  
    HttpSession session = request.getSession();  
    String originalSessionId = session.getId();  
    this.logger.debug(LogMessage.of(() -> "Invalidating session with Id '" + originalSessionId + "' "  
          + (this.migrateSessionAttributes ? "and" : "without") + " migrating attributes."));  
    Map<String, Object> attributesToMigrate = extractAttributes(session);  
    int maxInactiveIntervalToMigrate = session.getMaxInactiveInterval();  
    session.invalidate(); 
    //属性为true，即创建一个新的session 
    session = request.getSession(true); // we now have a new session  
    this.logger.debug(LogMessage.format("Started new session: %s", session.getId()));  
    transferAttributes(attributesToMigrate, session);  
    if (this.migrateSessionAttributes) {  
       session.setMaxInactiveInterval(maxInactiveIntervalToMigrate);  
    }  
    return session;  //返回新的session
}
```

创建了新的session从而改变sessionId
## SessionRegistry

存储SessionInformation

```Java
public interface SessionRegistry {  
  
    List<Object> getAllPrincipals();  
    List<SessionInformation> getAllSessions(Object principal, boolean includeExpiredSessions);  
  
    SessionInformation getSessionInformation(String sessionId);  
    void refreshLastRequest(String sessionId);  
  
    void registerNewSession(String sessionId, Object principal);  
  
    void removeSessionInformation(String sessionId);  
  
}
```

仅有一个实现类SessionRegistryImpl

## SessionInformation

维护了一些基本信息

```Java
public class SessionInformation implements Serializable {  
  
    private static final long serialVersionUID = SpringSecurityCoreVersion.SERIAL_VERSION_UID;  
  
    private Date lastRequest;  
  
    private final Object principal;  
  
    private final String sessionId;  
  
    private boolean expired = false;  
  
    public SessionInformation(Object principal, String sessionId, Date lastRequest) {  
       Assert.notNull(principal, "Principal required");  
       Assert.hasText(sessionId, "SessionId required");  
       Assert.notNull(lastRequest, "LastRequest required");  
       this.principal = principal;  
       this.sessionId = sessionId;  
       this.lastRequest = lastRequest;  
    }  
  
    public void expireNow() {  
       this.expired = true;  
    }  
  
    public Date getLastRequest() {  
       return this.lastRequest;  
    }  
  
    public Object getPrincipal() {  
       return this.principal;  
    }  
  
    public String getSessionId() {  
       return this.sessionId;  
    }  
  
    public boolean isExpired() {  
       return this.expired;  
    }  

	//每次刷新
    public void refreshLastRequest() {  
       this.lastRequest = new Date();  
    }  
  
}
```

## SessionInformationExpiredStrategy

```
/**  
 * Determines the behaviour of the ConcurrentSessionFilter when an expired session is detected in the ConcurrentSessionFilter. */
	
	public interface SessionInformationExpiredStrategy {  
    void onExpiredSessionDetected(SessionInformationExpiredEvent event) throws IOException, ServletException;  
  
}
```
决定[[Spring/Spring_Security_02#ConcurrentSessionFilter\|ConcurrentSessionFilter]]的过期策略

### onExpiredSessionDetected方法

![Pasted image 20231113113309.png|undefined](/img/user/Pasted%20image%2020231113113309.png)

#### ResponseBodySessionInformationExpiredStrategy中的实现
```Java
private static final class ResponseBodySessionInformationExpiredStrategy  
       implements SessionInformationExpiredStrategy {  
  
    @Override  
    public void onExpiredSessionDetected(SessionInformationExpiredEvent event) throws IOException {  
       HttpServletResponse response = event.getResponse();  
       response.getWriter()  
          .print("This session has been expired (possibly due to multiple concurrent "  
                + "logins being attempted as the same user).");  
       response.flushBuffer();  
    }  
  
}
```

#### SimpleRedirectSessionInformationExpiredStrategy中的实现
```Java
public final class SimpleRedirectSessionInformationExpiredStrategy implements SessionInformationExpiredStrategy {  
  
    private final Log logger = LogFactory.getLog(getClass());  
  
    private final String destinationUrl;  
  
    private final RedirectStrategy redirectStrategy;  
  
    public SimpleRedirectSessionInformationExpiredStrategy(String invalidSessionUrl) {  
       this(invalidSessionUrl, new DefaultRedirectStrategy());  
    }  
  
    public SimpleRedirectSessionInformationExpiredStrategy(String invalidSessionUrl,  
          RedirectStrategy redirectStrategy) {  
       Assert.isTrue(UrlUtils.isValidRedirectUrl(invalidSessionUrl), "url must start with '/' or with 'http(s)'");  
       this.destinationUrl = invalidSessionUrl;  
       this.redirectStrategy = redirectStrategy;  
    }  
  
    @Override  
    public void onExpiredSessionDetected(SessionInformationExpiredEvent event) throws IOException {  
       this.logger.debug("Redirecting to '" + this.destinationUrl + "'");  
       this.redirectStrategy.sendRedirect(event.getRequest(), event.getResponse(), this.destinationUrl);  
    }  
  
}
```

##### this.redirectStrategy.sendRedirect

重定向策略接口RedirectStrategy
```Java
public interface RedirectStrategy {  
  
    /**  
     * Performs a redirect to the supplied URL     * @param request the current request  
     * @param response the response to redirect  
     * @param url the target URL to redirect to, for example "/login"  
     */    
     void sendRedirect(HttpServletRequest request, HttpServletResponse response, String url) throws IOException;  
  
}
```
默认实现类为DefaultRedirectStrategy

### ConcurrentSessionFilter

doFilter方法

```Java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)  
       throws IOException, ServletException {  
    doFilter((HttpServletRequest) request, (HttpServletResponse) response, chain);  
}  
  
private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain) 
       throws IOException, ServletException {  
    HttpSession session = request.getSession(false);  
    if (session != null) {  
	    //通过sessionRegistry.getSessionInformation得到session信息
       SessionInformation info = this.sessionRegistry.getSessionInformation(session.getId());  
       if (info != null) {  
          if (info.isExpired()) {  
             // Expired - abort processing  
             this.logger.debug(LogMessage  
                .of(() -> "Requested session ID " + request.getRequestedSessionId() + " has expired."));  
             doLogout(request, response);  
             //2️⃣
             this.sessionInformationExpiredStrategy  
                .onExpiredSessionDetected(new SessionInformationExpiredEvent(info, request, response));  
             return;  
          }  
          // Non-expired - update last request date/time  
          this.sessionRegistry.refreshLastRequest(info.getSessionId());  
       }  
    }  
    chain.doFilter(request, response);  
}
```

2️⃣this.sessionInformationExpiredStrategy.onExpiredSessionDetected
如果过期，就调用策略类的处理过期的方法onExpiredSessionDetected，该方法由其实现类具体实现

## SessionManagementConfigurer

![Pasted image 20231113191906.png|undefined](/img/user/Pasted%20image%2020231113191906.png)

重要方法：init和configure

```Java
@Override  
public void init(H http) {  //set一些属性
    SecurityContextRepository securityContextRepository = http.getSharedObject(SecurityContextRepository.class);  
    boolean stateless = isStateless();  
    if (securityContextRepository == null) {  
       if (stateless) {  
          http.setSharedObject(SecurityContextRepository.class, new RequestAttributeSecurityContextRepository());  
          this.sessionManagementSecurityContextRepository = new NullSecurityContextRepository();  
       }  
       else {  
          HttpSessionSecurityContextRepository httpSecurityRepository = new HttpSessionSecurityContextRepository();  
          httpSecurityRepository.setDisableUrlRewriting(!this.enableSessionUrlRewriting);  
          httpSecurityRepository.setAllowSessionCreation(isAllowSessionCreation());  
          AuthenticationTrustResolver trustResolver = http.getSharedObject(AuthenticationTrustResolver.class);  
          if (trustResolver != null) {  
             httpSecurityRepository.setTrustResolver(trustResolver);  
          }  
          this.sessionManagementSecurityContextRepository = httpSecurityRepository;  
          DelegatingSecurityContextRepository defaultRepository = new DelegatingSecurityContextRepository(  
                httpSecurityRepository, new RequestAttributeSecurityContextRepository());  
          http.setSharedObject(SecurityContextRepository.class, defaultRepository);  
       }  
    }  
    else {  
       this.sessionManagementSecurityContextRepository = securityContextRepository;  
    }  
    RequestCache requestCache = http.getSharedObject(RequestCache.class);  
    if (requestCache == null) {  
       if (stateless) {  
          http.setSharedObject(RequestCache.class, new NullRequestCache());  
       }  
    }  
    http.setSharedObject(SessionAuthenticationStrategy.class, getSessionAuthenticationStrategy(http));  
    http.setSharedObject(InvalidSessionStrategy.class, getInvalidSessionStrategy());  
}  
  
@Override  
public void configure(H http) {  //添加sessoin相关的filter
    SessionManagementFilter sessionManagementFilter = createSessionManagementFilter(http);  
    if (sessionManagementFilter != null) {  
       http.addFilter(sessionManagementFilter);  
    }  
    if (isConcurrentSessionControlEnabled()) {  
       ConcurrentSessionFilter concurrentSessionFilter = createConcurrencyFilter(http);  
  
       concurrentSessionFilter = postProcess(concurrentSessionFilter);  
       http.addFilter(concurrentSessionFilter);  
    }  
    if (!this.enableSessionUrlRewriting) {  
       http.addFilter(new DisableEncodeUrlFilter());  
    }  
    if (this.sessionPolicy == SessionCreationPolicy.ALWAYS) {  
       http.addFilter(new ForceEagerSessionCreationFilter());  
    }  
}  
```

---

# 一些Filter

DefaultSecurityFilterChain中的filter包含：
![Pasted image 20231113194321.png|undefined](/img/user/Pasted%20image%2020231113194321.png)


## DisableEncodeUrlFilter

```Java
public class DisableEncodeUrlFilter extends OncePerRequestFilter {  
  
    @Override  
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)  
          throws ServletException, IOException {  
       filterChain.doFilter(request, new 1️⃣DisableEncodeUrlResponseWrapper(response));  
    }  

	  
  
}
```

### 1️⃣DisableEncodeUrlResponseWrapper

new DisableEncodeUrlResponseWrapper(response)：
该参数是一个response的包装

![Pasted image 20231113195339.png|undefined](/img/user/Pasted%20image%2020231113195339.png)


```Java
	  //1️⃣
      private static final class DisableEncodeUrlResponseWrapper extends HttpServletResponseWrapper {  
  
       private DisableEncodeUrlResponseWrapper(HttpServletResponse response) {  
          super(response);  
       }  
  
       @Override  
       public String encodeRedirectURL(String url) {  
          return url;  
       }  

		//2️⃣把sessionId编码到url中
       @Override  
       public String encodeURL(String url) {  
          return url;  
       }  
  
    }  
```


>装饰器模式：类中先维护一份被装饰的对象，然后实现被装饰对象的方法，最后默认调用被装饰对象的同名方法
>比如该类的父类HttpServletResponseWrapper，还有HttpServletRequestWrapper

#### 2️⃣encodeURL

在HttpServletResponse中的声明：
```Java
/**  
 Encodes the specified URL by including the session ID in it, or, if encoding is not needed, returns the URL unchanged. The implementation of this method includes the logic to determine whether the session ID needs to be encoded in the URL. For example, if the browser supports cookies, or session tracking is turned off, URL encoding is unnecessary.
For robust session tracking, all URLs emitted by a servlet should be run through this method. Otherwise, URL rewriting cannot be used with browsers which do not support cookies.
Params:
url – the url to be encoded.
Returns:
the encoded URL if encoding is needed; the unchanged URL otherwise. */
	String encodeURL(String url);
```
如果有sessionId，就将sessionId包含在url中并返回；如果没有sessionId，就直接返回url
而本类直接重写了encodeURL方法，直接返回url


**所以DisableEncodeUrlFilter的作用就是让这些包含sessionId到url的方法失效**


## WebAsyncManagerIntegrationFilter

管理异步执行，在异步执行之前把SecurityContext放进去
```Java
public final class WebAsyncManagerIntegrationFilter extends OncePerRequestFilter {  
  
    private static final Object CALLABLE_INTERCEPTOR_KEY = new Object();  
  
    private SecurityContextHolderStrategy securityContextHolderStrategy = SecurityContextHolder  
       .getContextHolderStrategy();  
  
    @Override  
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)  
          throws ServletException, IOException {  
       WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);  
       SecurityContextCallableProcessingInterceptor securityProcessingInterceptor = (SecurityContextCallableProcessingInterceptor) asyncManager  
          .getCallableInterceptor(CALLABLE_INTERCEPTOR_KEY);  
       if (securityProcessingInterceptor == null) {  
          SecurityContextCallableProcessingInterceptor interceptor = new SecurityContextCallableProcessingInterceptor();  //1️⃣
          interceptor.setSecurityContextHolderStrategy(this.securityContextHolderStrategy);  
          asyncManager.registerCallableInterceptor(CALLABLE_INTERCEPTOR_KEY, interceptor);  
       }  
       filterChain.doFilter(request, response);  
    }  
  
    public void setSecurityContextHolderStrategy(SecurityContextHolderStrategy securityContextHolderStrategy) {  
       Assert.notNull(securityContextHolderStrategy, "securityContextHolderStrategy cannot be null");  
       this.securityContextHolderStrategy = securityContextHolderStrategy;  
    }  
  
}
```

### 1️⃣SecurityContextCallableProcessingInterceptor

```Java
public final class SecurityContextCallableProcessingInterceptor implements CallableProcessingInterceptor {  
  
    private volatile SecurityContext securityContext;  
  
    private SecurityContextHolderStrategy securityContextHolderStrategy = SecurityContextHolder  
       .getContextHolderStrategy();  
  
    public SecurityContextCallableProcessingInterceptor() {  
    }  
  
    public SecurityContextCallableProcessingInterceptor(SecurityContext securityContext) {  
       Assert.notNull(securityContext, "securityContext cannot be null");  
       setSecurityContext(securityContext);  
    }  
  
    @Override  
    public <T> void beforeConcurrentHandling(NativeWebRequest request, Callable<T> task) {  
       if (this.securityContext == null) {  
          setSecurityContext(this.securityContextHolderStrategy.getContext());  
       }  
    }  
  
    @Override  
    //在真正执行异步任务之前，会首先执行该拦截器，就会执行preProcess方法，从而执行setContext设置context
    //后续就可以通过getContext方法来获得用户信息
    public <T> void preProcess(NativeWebRequest request, Callable<T> task) {  
       this.securityContextHolderStrategy.setContext(this.securityContext);  
    }  
  
    @Override  
    public <T> void postProcess(NativeWebRequest request, Callable<T> task, Object concurrentResult) {  
       this.securityContextHolderStrategy.clearContext();  
    }  
  
    public void setSecurityContextHolderStrategy(SecurityContextHolderStrategy securityContextHolderStrategy) {  
       Assert.notNull(securityContextHolderStrategy, "securityContextHolderStrategy cannot be null");  
       this.securityContextHolderStrategy = securityContextHolderStrategy;  
    }  
  
    private void setSecurityContext(SecurityContext securityContext) {  
       this.securityContext = securityContext;  
    }  
  
}
```

## HeaderWriterFilter

写响应头，委派给HeaderWriter接口，有很多实现类

```Java
public class HeaderWriterFilter extends OncePerRequestFilter {  
	//维护了很多HeaderWriter
	private final List<HeaderWriter> headerWriters;

	......
```

```Java
public interface HeaderWriter {  
    void writeHeaders(HttpServletRequest request, HttpServletResponse response);  
}
```
