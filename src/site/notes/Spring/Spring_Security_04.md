---
{"dg-publish":true,"permalink":"/spring/spring-security-04/","dgPassFrontmatter":true}
---

# 关于授权的过滤器

- AuthorizationFilter：基于类的授权
- FilterSecurityInterceptor：基于表达式的授权

# AuthorizationFilter及相关类

* AuthorizationFilter

* AuthorizationManager

* AuthorizationDecision

* AuthorizeHttpRequestsConfigurer

* RequestAuthorizationContext

* AuthorizationManagerRequestMatcherRegistry

* AbstractRequestMatcherRegistry

* AuthorizedUrl

AuthorizationFilter由AuthorizeHttpRequestsConfigurer构建出来，
属性通过AuthorizeHttpRequestsConfigurer去set进filter，
最终由configurer将filter添加进过滤器链

## AuthorizationFilter

### doFilter方法

```Java
@Override  
public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain chain)  
       throws ServletException, IOException {  
  
    HttpServletRequest request = (HttpServletRequest) servletRequest;  
    HttpServletResponse response = (HttpServletResponse) servletResponse;  
  
    if (this.observeOncePerRequest && isApplied(request)) {  
       chain.doFilter(request, response);  
       return;  
    }  
  
    if (skipDispatch(request)) {  
       chain.doFilter(request, response);  
       return;  
    }  
  
    String alreadyFilteredAttributeName = getAlreadyFilteredAttributeName();  
    request.setAttribute(alreadyFilteredAttributeName, Boolean.TRUE);  
    try {  
	    //得到授权结果
       AuthorizationDecision decision = this.authorizationManager.check(this::getAuthentication, request);  
       this.eventPublisher.publishAuthorizationEvent(this::getAuthentication, request, decision);  
       if (decision != null && !decision.isGranted()) {  //得不到抛异常
          throw new AccessDeniedException("Access Denied");  
       }  
       chain.doFilter(request, response);  
    }  
    finally {  
       request.removeAttribute(alreadyFilteredAttributeName);  
    }  
}
```

利用[[Spring/Spring_Security_04#AuthorizationManager\|authorizationManager]]的[[Spring/Spring_Security_04#check方法\|check]]方法校验

### getAuthentication方法
```Java
private Authentication getAuthentication() {  
    Authentication authentication = this.securityContextHolderStrategy.getContext().getAuthentication();  
    if (authentication == null) {  
       throw new AuthenticationCredentialsNotFoundException(  
             "An Authentication object was not found in the SecurityContext");  
    }  
    return authentication;  
}
```
## AuthorizationManager

```
//决策能不能访问T类型的对象
An Authorization manager which can determine if an Authentication has access to a specific object.
Type parameters:
<T> – the type of object that the authorization check is being done on.
```

```Java
@FunctionalInterface  
public interface AuthorizationManager<T> {  
  
    default void verify(Supplier<Authentication> authentication, T object) {  
       AuthorizationDecision decision = check(authentication, object);  
       if (decision != null && !decision.isGranted()) {  
          throw new AccessDeniedException("Access Denied");  
       }  
    }  
  
    @Nullable  //可以返回null
    AuthorizationDecision check(Supplier<Authentication> authentication, T object);  
}
```

### check方法

Supplier<\Authentication> authentication：
提供器，可以得到一个authentication，
在[[Spring/Spring_Security_04#AuthorizationFilter\|AuthorizationFilter]]类的[[Spring/Spring_Security_04#doFilter方法\|doFilter]]方法中被调用：this.authorizationManager.check(this::getAuthentication, request)
因此改authentication为[[Spring/Spring_Security_04#AuthorizationFilter\|AuthorizationFilter]]中[[Spring/Spring_Security_04#getAuthentication方法\|getAuthentication]]方法得到的Authentication

返回值为[[Spring/Spring_Security_04#AuthorizationDecision\|AuthorizationDecision]]
## AuthorizationDecision

```Java
public class AuthorizationDecision {  
  
    private final boolean granted;  
  
    public AuthorizationDecision(boolean granted) {  
       this.granted = granted;  
    }  
  
    public boolean isGranted() {  
       return this.granted;  
    }  
  
    @Override  
    public String toString() {  
       return getClass().getSimpleName() + " [granted=" + this.granted + "]";  
    }  
  
}
```

将granted作为一个类AuthorizationDecision的属性而不是直接让[[Spring/Spring_Security_04#check方法\|check]]方法返回一个bool值，是为了check方法的可拓展性


## AuthorizeHttpRequestsConfigurer

![Pasted image 20231119193335.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231119193335.png)

### 内部类AuthorizedUrl的configure方法

```Java
@Override  
public void configure(H http) {  
	//🐱
    AuthorizationManager<HttpServletRequest> authorizationManager = this.registry.createAuthorizationManager();  
    AuthorizationFilter authorizationFilter = new AuthorizationFilter(authorizationManager);  
    authorizationFilter.setAuthorizationEventPublisher(this.publisher);  
    authorizationFilter.setShouldFilterAllDispatcherTypes(this.registry.shouldFilterAllDispatcherTypes);  
    authorizationFilter.setSecurityContextHolderStrategy(getSecurityContextHolderStrategy());  
    http.addFilter(postProcess(authorizationFilter));  
}
```

🐱
通过registry的createAuthorizationManager方法得到一个AuthorizationManager<\HttpServletRequest>
registry的定义为：private final [[Spring/Spring_Security_04#AuthorizationManagerRequestMatcherRegistry\|AuthorizationManagerRequestMatcherRegistry registry]]

### addMapping方法

```Java
private AuthorizationManagerRequestMatcherRegistry addMapping(List<? extends RequestMatcher> matchers, AuthorizationManager<RequestAuthorizationContext> manager) {  
    for (RequestMatcher matcher : matchers) {  
	    //循环得到RequestMatcher，调用registry的addMapping方法
       this.registry.addMapping(matcher, manager);  
    }  
    return this.registry;  
}
```

registry.addMapping(matcher, manager)：
registry为[[Spring/Spring_Security_04#AuthorizationManagerRequestMatcherRegistry\|AuthorizationManagerRequestMatcherRegistry]]类，含有[[Spring/Spring_Security_04#addMapping\|addMapping]]方法


### 内部类AuthorizedUrl

#### access方法

```Java
public AuthorizationManagerRequestMatcherRegistry access(  
       AuthorizationManager<RequestAuthorizationContext> manager) {  
    Assert.notNull(manager, "manager cannot be null");  
    return AuthorizeHttpRequestsConfigurer.this.addMapping(this.matchers, manager);  
}
```

> 参数为AuthorizationManager<RequestAuthorizationContext\>：
>关于[[Spring/Spring_Security_04#RequestAuthorizationContext\|RequestAuthorizationContext]]类


>AuthorizeHttpRequestsConfigurer.this.addMapping(this.matchers, manager)：
>通过AuthorizeHttpRequestsConfigurer.this得到父类AuthorizeHttpRequestsConfigurer的对象，然后调用父类的[[Spring/Spring_Security_04#addMapping方法\|addMapping]]方法



## AuthorizationManagerRequestMatcherRegistry

为[[Spring/Spring_Security_04#AuthorizeHttpRequestsConfigurer\|AuthorizeHttpRequestsConfigurer]]的内部类

```Java
public final class AuthorizationManagerRequestMatcherRegistry 
extends AbstractRequestMatcherRegistry<AuthorizedUrl>
```

继承了[[Spring/Spring_Security_04#父类AbstractRequestMatcherRegistry\|AbstractRequestMatcherRegistry]]，泛型C为[[Spring/Spring_Security_04#内部类AuthorizedUrl\|AuthorizedUrl]]

![Pasted image 20231119195801.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231119195801.png)

### 父类AbstractRequestMatcherRegistry

```
A base class for registering RequestMatcher's. For example, it might allow for specifying which RequestMatcher require a certain level of authorization.
```

一个匹配器要匹配什么请求 这个请求需要什么样的授权处理
#### requestMatchers方法
```Java
public C requestMatchers(RequestMatcher... requestMatchers) {  
    Assert.state(!this.anyRequestConfigured, "Can't configure requestMatchers after anyRequest");  
    return chainRequestMatchers(Arrays.asList(requestMatchers));  
}  

  
public C requestMatchers(HttpMethod method, String... patterns) {  
    if (!mvcPresent) {  
       return requestMatchers(RequestMatchers.antMatchersAsArray(method, patterns));  
    }  
    if (!(this.context instanceof WebApplicationContext)) {  
       return requestMatchers(RequestMatchers.antMatchersAsArray(method, patterns));  
    }  
    WebApplicationContext context = (WebApplicationContext) this.context;  
    ServletContext servletContext = context.getServletContext();  
    if (servletContext == null) {  
       return requestMatchers(RequestMatchers.antMatchersAsArray(method, patterns));  
    }  
    Map<String, ? extends ServletRegistration> registrations = mappableServletRegistrations(servletContext);  
    if (registrations.isEmpty()) {  
       return requestMatchers(RequestMatchers.antMatchersAsArray(method, patterns));  
    }  
    if (!hasDispatcherServlet(registrations)) {  
       return requestMatchers(RequestMatchers.antMatchersAsArray(method, patterns));  
    }  
    if (registrations.size() > 1) {  
       String errorMessage = computeErrorMessage(registrations.values());  
       throw new IllegalArgumentException(errorMessage);  
    }  
    return requestMatchers(createMvcMatchers(method, patterns).toArray(new RequestMatcher[0]));  
}
```

该方法构建出一个或多个[[Spring/Spring_Security_04#内部类RequestMatchers\|RequestMatchers]]实例，并将对象实例传给chainRequestMatchers

#### 泛型C

public abstract class AbstractRequestMatcherRegistry<\C>

```
The object that is returned or Chained after creating the RequestMatcher
```

返回一个对象或者这个对象可以往下链式调用

#### chainRequestMatchers抽象方法
```Java
protected abstract C chainRequestMatchers(List<RequestMatcher> requestMatchers);
```

该方法的实现类：
![Pasted image 20231119205208.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231119205208.png)

[[Spring/Spring_Security_04#chainRequestMatchers方法\|在AuthorizeHttpRequestsConfigurer中的实现]]

#### 内部类RequestMatchers

antMatchers方法
```Java
static List<RequestMatcher> antMatchers(HttpMethod httpMethod, String... antPatterns) {  
    return Arrays.asList(antMatchersAsArray(httpMethod, antPatterns));  
}  
  
static List<RequestMatcher> antMatchers(String... antPatterns) {  
    return antMatchers(null, antPatterns);  
}  
  
static RequestMatcher[] antMatchersAsArray(HttpMethod httpMethod, String... antPatterns) {  
    String method = (httpMethod != null) ? httpMethod.toString() : null;  
    // new出来RequestMatcher
    RequestMatcher[] matchers = new RequestMatcher[antPatterns.length];  
    for (int index = 0; index < antPatterns.length; index++) {  
       matchers[index] = new AntPathRequestMatcher(antPatterns[index], method);  
    }  
    return matchers;  
}
```


### chainRequestMatchers方法
```Java
@Override  
protected AuthorizedUrl chainRequestMatchers(List<RequestMatcher> requestMatchers) { 
	//用类变量unmappedMatchers保存一下
    this.unmappedMatchers = requestMatchers;  
    //返回一个保存着requestMatchers的AuthorizedUrl对象
    return new AuthorizedUrl(requestMatchers);  
}
```

返回一个[[Spring/Spring_Security_04#内部类AuthorizedUrl\|AuthorizedUrl]]的实例

### addMapping

```Java
private void addMapping(RequestMatcher matcher, AuthorizationManager<RequestAuthorizationContext> manager) {  
    this.unmappedMatchers = null;  
    this.managerBuilder.add(matcher, manager);  
    this.mappingCount++;  
}
```

调用managerBuilder的add方法，在managerBuilder中维护了一堆映射

属性managerBuilder的定义：
```Java
private final RequestMatcherDelegatingAuthorizationManager.Builder managerBuilder = RequestMatcherDelegatingAuthorizationManager  
    .builder();
```
关于[[Spring/Spring_Security_04#RequestMatcherDelegatingAuthorizationManager\|RequestMatcherDelegatingAuthorizationManager]]类及其内部类[[Spring/Spring_Security_04#内部类Builder\|Builder]]

## RequestMatcherDelegatingAuthorizationManager

### 属性mappings
```Java
private final List<RequestMatcherEntry<AuthorizationManager<RequestAuthorizationContext>>> mappings;
```

mappings List泛型为[[Spring/Spring_Security_04#RequestMatcherEntry\|RequestMatcherEntry]]，
[[Spring/Spring_Security_04#RequestMatcherEntry\|RequestMatcherEntry]]的泛型为[[Spring/Spring_Security_04#AuthorizationManager\|AuthorizationManager]]，即[[Spring/Spring_Security_04#RequestMatcherEntry\|RequestMatcherEntry]]中的entry属性为[[Spring/Spring_Security_04#AuthorizationManager\|AuthorizationManager]]，
AuthorizationManager泛型为[[Spring/Spring_Security_04#RequestAuthorizationContext\|RequestAuthorizationContext]]。

通过构造方法传入
```Java
private RequestMatcherDelegatingAuthorizationManager(  
       List<RequestMatcherEntry<AuthorizationManager<RequestAuthorizationContext>>> mappings) {  
    Assert.notEmpty(mappings, "mappings cannot be empty");  
    this.mappings = mappings;  
}
```

### 内部类Builder

```Java
public static final class Builder {  
	//维护了和父类一样的mappings
    private final List<RequestMatcherEntry<AuthorizationManager<RequestAuthorizationContext>>> mappings = new ArrayList<>();  

	//add方法直接new一个RequestMatcherEntry加入mappings
   public Builder add(RequestMatcher matcher, AuthorizationManager<RequestAuthorizationContext> manager) {  
       Assert.notNull(matcher, "matcher cannot be null");  
       Assert.notNull(manager, "manager cannot be null");  
       this.mappings.add(new RequestMatcherEntry<>(matcher, manager));  
       return this;  
    }  

	//通过使用消费器增加mappings的自定义程度
    public Builder mappings(  
          Consumer<List<RequestMatcherEntry<AuthorizationManager<RequestAuthorizationContext>>>> mappingsConsumer) {  
       Assert.notNull(mappingsConsumer, "mappingsConsumer cannot be null");  
       mappingsConsumer.accept(this.mappings);  
       return this;  
    }  
    
    public RequestMatcherDelegatingAuthorizationManager build() {  
       return new RequestMatcherDelegatingAuthorizationManager(this.mappings);  
    }  
  
}
```

## RequestMatcherEntry

```Java
public class RequestMatcherEntry<T> {  
	//维护了两个属性
  
    private final RequestMatcher requestMatcher;  
  
    private final T entry;  
  
    public RequestMatcherEntry(RequestMatcher requestMatcher, T entry) {  
       this.requestMatcher = requestMatcher;  
       this.entry = entry;  
    }  
  
    public RequestMatcher getRequestMatcher() {  
       return this.requestMatcher;  
    }  
  
    public T getEntry() {  
       return this.entry;  
    }  
  
}
```

## RequestAuthorizationContext

维护了两个属性：
```Java
private final HttpServletRequest request;  
  
private final Map<String, String> variables;
```


## 🌟授权过程

### mappings的构建

mapping在添加的过程中，最终会被维护到managerBuilder（属性定义：[[Spring/Spring_Security_04#RequestMatcherDelegatingAuthorizationManager\|RequestMatcherDelegatingAuthorizationManager]].[[Spring/Spring_Security_04#内部类Builder\|Builder]] managerBuilder）中

[[Spring/Spring_Security_04#AuthorizeHttpRequestsConfigurer\|AuthorizeHttpRequestsConfigurer]]中维护了属性registry，该属性为AuthorizationManagerRequestMatcherRegistry
[[Spring/Spring_Security_04#父类AbstractRequestMatcherRegistry\|AbstractRequestMatcherRegistry]]为AuthorizationManagerRequestMatcherRegistry的父类，维护了用于创建请求匹配器的方法，最终执行到chainRequestMatchers方法，返回一个保存着List<\RequestMatcher> requestMatchers的[[Spring/Spring_Security_04#内部类AuthorizedUrl\|AuthorizedUrl]]对象

在AuthorizedUrl类中，通过hasAnyRole方法得到一个AuthorityAuthorizationManager传给access方法，
然后access方法通过`return AuthorizeHttpRequestsConfigurer.this.addMapping(this.matchers,manager);`调用AuthorizeHttpRequestsConfigurer的addMapping方法

AuthorizeHttpRequestsConfigurer的addMapping方法通过遍历每个RequestMatcher并调用this.registry的addMapping方法（AuthorizationManagerRequestMatcherRegistry类中）

在this.registry的addMapping方法中，通过调用this.managerBuilder.add(matcher, manager)，执行Builder类（RequestMatcherDelegatingAuthorizationManager的内部类）的add方法，添加至Builder类的属性`private final List<RequestMatcherEntry<AuthorizationManager<\RequestAuthorizationContext>>> mappings = new ArrayList<>();

### 将mappings添加到RequestMatcherDelegatingAuthorizationManager

在AuthorizedUrl的configure方法中，
执行this.registry.createAuthorizationManager()，
进入该类的createAuthorizationManager方法，执行postProcess(this.managerBuilder.build())，
进入Builder类的build方法，`return new RequestMatcherDelegatingAuthorizationManager(this.mappings)
而RequestMatcherDelegatingAuthorizationManager恰好需要一个mappings

所以最终得到的authorizationManager的类型是[[Spring/Spring_Security_04#RequestMatcherDelegatingAuthorizationManager\|RequestMatcherDelegatingAuthorizationManager]]，该类是真正做授权过程的一个类。


### 为什么要维护mappings

SpringSecurity定义了一个接口AuthorizationManager<\T>，通过其中的check方法得到一个AuthorizationDecision授权结果，根据结果判断是否有权限，
不同的请求使用不同的requestMatcher来匹配，
匹配好之后用AuthorizationManager的一个实现类AuthorityAuthorizationManager<\T>类真正去判断请求有没有权限

因为不同的请求有不同的manager，所以需要维护很多request到AuthorizationManager的一个映射，
该映射通过类`RequestMatcherDelegatingAuthorizationManager implements AuthorizationManager<HttpServletRequest\>`中的mappings属性来维护


### RequestMatcherDelegatingAuthorizationManager在AuthorizationFilter被维护

在AuthorizationFilter的doFilter方法中执行到`this.authorizationManager.check(this::getAuthentication, request);`时，最终会调用RequestMatcherDelegatingAuthorizationManager中的check方法并将结果返回到doFilter方法中用`AuthorizationDecision decision`去接收，然后执行后面的处理逻辑

### RequestMatcherDelegatingAuthorizationManage类中check方法的逻辑

真正执行授权逻辑的为RequestMatcherDelegatingAuthorizationManage类

该类的check方法通过遍历mappings拿到每一个mapping的requestMatcher，然后用requestMatcher来匹配request
如果匹配就通过mapping.getEntry拿到真正用来干活的`AuthorizationManager<RequestAuthorizationContext> manager`，然后调用`manager. check(authentication,new RequestAuthorizationContext (request, matchResult.getVariables()))`，其中matchResult.getVariables()为请求里面的模版变量
如果不匹配就返回null


## 配置


```
http.authorizeHttpRequests(author -> author.requestMatchers("/admin/**").hasRole("ADMIN"))
```

config类中通过httpSecurity.authorizeHttpRequest()得到AuthorizeHttpRequestsConfigurer的registry，
```Java
@Deprecated(since = "6.1", forRemoval = true)  
public AuthorizeHttpRequestsConfigurer<HttpSecurity>.AuthorizationManagerRequestMatcherRegistry authorizeHttpRequests()  
       throws Exception {  
    ApplicationContext context = getContext();  
    //得到registry，即AuthorizationManagerRequestMatcherRegistry
    return getOrApply(new AuthorizeHttpRequestsConfigurer<>(context)).getRegistry();  
}
```
然后我们就可以调用AuthorizationManagerRequestMatcherRegistry中的方法，
例如.antMatchers，此时返回一个AuthorizedUrl，我们可以继续调用AuthorizedUrl中的方法，
例如hasRole，此时又回返回一个registry

---

# FilterSecurityInterceptor及其相关类

* FilterSecurityInterceptor

* AbstractSecurityInterceptor

* FilterInvocation

* ConfigAttribute

* SecurityConfig

* WebExpressionConfigAttribute

* SecurityExpressionHandler

* AbstractInterceptUrlConfigurer

* UrlAuthorizationConfigurer

* ExpressionUrlAuthorizationConfigurer

* AbstractInterceptUrlRegistry

* AccessDecisionManager

* AccessDecisionVoter

* AffirmativeBased

## FilterSecurityInterceptor

![Pasted image 20231121201823.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231121201823.png)

### doFilter方法

```Java
@Override  
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)  
       throws IOException, ServletException {  
    invoke(new FilterInvocation(request, response, chain));  
}
```

### invoke方法

```Java
public void invoke(FilterInvocation filterInvocation) throws IOException, ServletException {  
    if (isApplied(filterInvocation) && this.observeOncePerRequest) {  
       filterInvocation.getChain().doFilter(filterInvocation.getRequest(), filterInvocation.getResponse());  
       return;  
    }  
    // first time this request being called, so perform security checking  
    if (filterInvocation.getRequest() != null && this.observeOncePerRequest) {  
       filterInvocation.getRequest().setAttribute(FILTER_APPLIED, Boolean.TRUE);  
    }  
    InterceptorStatusToken token = super.beforeInvocation(filterInvocation);  
    try {  
       filterInvocation.getChain().doFilter(filterInvocation.getRequest(), filterInvocation.getResponse());  
    }  
    finally {  
       super.finallyInvocation(token);  
    }  
    super.afterInvocation(token, null);  
}
```

#### [[Spring/Spring_Security_04#beforeInvocation方法\|beforeInvocation]]方法

该方法的核心逻辑。调用父类[[Spring/Spring_Security_04#AbstractSecurityInterceptor\|AbstractSecurityInterceptor]]的[[Spring/Spring_Security_04#beforeInvocation方法\|beforeInvocation(filterInvocation)]]方法，返回一个InterceptorStatusToken token

#### [[Spring/Spring_Security_04#finallyInvocation方法\|finallyInvocation]]方法
#### [[Spring/Spring_Security_04#afterInvocation方法\|afterInvocation]]方法

#### obtainSecurityMetadataSource方法

```Java
private FilterInvocationSecurityMetadataSource securityMetadataSource;

public SecurityMetadataSource obtainSecurityMetadataSource() {  
    return this.securityMetadataSource;  
}
```


##### FilterInvocationSecurityMetadataSource

```Java
public interface FilterInvocationSecurityMetadataSource extends SecurityMetadataSource {  
  
}
```

实现类
![Pasted image 20231126160103.png|undefined](/img/user/Pasted%20image%2020231126160103.png)

## AbstractSecurityInterceptor

### beforeInvocation方法
```Java
//Object为安全对象
protected InterceptorStatusToken beforeInvocation(2️⃣Object object) {  
    Assert.notNull(object, "Object was null");  
    //如果拿到的不是FilterInvocation，就直接报错
    if (!getSecureObjectClass().isAssignableFrom(object.getClass())) {  
       throw new IllegalArgumentException("Security invocation attempted for object " + object.getClass().getName()  
             + " but AbstractSecurityInterceptor only configured to support secure objects of type: "  
             + getSecureObjectClass());  
    } 
    //5️⃣   
    Collection<ConfigAttribute> attributes = this.obtainSecurityMetadataSource().getAttributes(object);  
    if (CollectionUtils.isEmpty(attributes)) {  
       Assert.isTrue(!this.rejectPublicInvocations,  
             () -> "Secure object invocation " + object  
                   + " was denied as public invocations are not allowed via this interceptor. "  
                   + "This indicates a configuration error because the "  
                   + "rejectPublicInvocations property is set to 'true'");  
       if (this.logger.isDebugEnabled()) {  
          this.logger.debug(LogMessage.format("Authorized public object %s", object));  
       }       publishEvent(new PublicInvocationEvent(object));  
       return null; // no further work post-invocation  
    }  
    if (this.securityContextHolderStrategy.getContext().getAuthentication() == null) {  
       credentialsNotFound(this.messages.getMessage("AbstractSecurityInterceptor.authenticationNotFound",  
             "An Authentication object was not found in the SecurityContext"), object, attributes);  
    }    
    //该方法：如果Authentication就去认证一下，返回一个认证后的Authentication
    Authentication authenticated = authenticateIfRequired();  
    if (this.logger.isTraceEnabled()) {  
       this.logger.trace(LogMessage.format("Authorizing %s with attributes %s", object, attributes));  
    }    // Attempt authorization  
    //4️⃣真正尝试授权的方法
    attemptAuthorization(object, attributes, authenticated);  
    if (this.logger.isDebugEnabled()) {  
       this.logger.debug(LogMessage.format("Authorized %s with attributes %s", object, attributes));  
    }    if (this.publishAuthorizationSuccess) {  
       publishEvent(new AuthorizedEvent(object, attributes, authenticated));  
    }  
    // Attempt to run as a different user  
    //1️⃣
    Authentication runAs = this.runAsManager.buildRunAs(authenticated, object, attributes);  
    if (runAs != null) { 
	    //替换我们的context 
       SecurityContext origCtx = this.securityContextHolderStrategy.getContext();  
       SecurityContext newCtx = this.securityContextHolderStrategy.createEmptyContext();  
       newCtx.setAuthentication(runAs);  
       this.securityContextHolderStrategy.setContext(newCtx);  
  
       if (this.logger.isDebugEnabled()) {  
          this.logger.debug(LogMessage.format("Switched to RunAs authentication %s", runAs));  
       }       // need to revert to token.Authenticated post-invocation  
       return new InterceptorStatusToken(origCtx, true, attributes, object);  
    }    this.logger.trace("Did not switch RunAs authentication since RunAsManager returned null");  
    // no further work post-invocation  
    //3️⃣
    return new InterceptorStatusToken(this.securityContextHolderStrategy.getContext(), false, attributes, object);  
  
}
```

#### 1️⃣RunAsManager
可以将我们的Authentication变为RunAsUserToken（根据传入的attributes信息构建），是一种拥有特殊权限或角色信息的Authentication，可以使我们变相的去访问某些资源
在FilterSecurityInterceptor的[[Spring/Spring_Security_04#invoke方法\|invoke]]方法的[[Spring/Spring_Security_04#finallyInvocation方法\|finallyInvocation]]中被复原回原来的Authentication

#### 2️⃣Object类型参数
Object类型参数为securityObject，即安全对象。此处为[[Spring/Spring_Security_04#FilterInvocation\|FilterInvocation]]

#### 3️⃣InterceptorStatusToken

```Java
public class InterceptorStatusToken {  
  
    private SecurityContext securityContext;  
  
    private Collection<ConfigAttribute> attr;  
	//安全对象
    private Object secureObject;  
	//SecurityContext如果被替换，则为true，未被替换则为false
    private boolean contextHolderRefreshRequired;  
    
	......
```

#### 4️⃣attemptAuthorization方法
```Java
private void attemptAuthorization(Object object, Collection<ConfigAttribute> attributes,  
       Authentication authenticated) {  
    try {  
	    //
       this.accessDecisionManager.decide(authenticated, object, attributes);  
    }    catch (AccessDeniedException ex) {  
       if (this.logger.isTraceEnabled()) {  
          this.logger.trace(LogMessage.format("Failed to authorize %s with attributes %s using %s", object,  
                attributes, this.accessDecisionManager));  
       }       else if (this.logger.isDebugEnabled()) {  
          this.logger.debug(LogMessage.format("Failed to authorize %s with attributes %s", object, attributes));  
       }       publishEvent(new AuthorizationFailureEvent(object, attributes, authenticated, ex));  
       throw ex;  
    }}
```

`this.accessDecisionManager.decide(authenticated, object, attributes);`

#### 5️⃣SecurityMetadataSource

`obtainSecurityMetadataSource`属性为SecurityMetadataSource类
存储一堆安全属性
```Java
public interface SecurityMetadataSource extends AopInfrastructureBean {  
	
	//获取跟Object相关的一些配置信息
    Collection<ConfigAttribute> getAttributes(Object object) throws IllegalArgumentException;  
  
    Collection<ConfigAttribute> getAllConfigAttributes();  

	//传一个类过来，判断支不支持返回这个类对应的配置信息。该类为getAttributes(Object object)中object的父类
    boolean supports(Class<?> clazz);  
  
}
```

##### 关于[[Spring/Spring_Security_04#ConfigAttribute\|ConfigAttribute]]

##### obtainSecurityMetadataSource抽象方法

```Java
public abstract SecurityMetadataSource obtainSecurityMetadataSource();
```

实现类
![Pasted image 20231126155522.png|undefined](/img/user/Pasted%20image%2020231126155522.png)
[[Spring/Spring_Security_04#obtainSecurityMetadataSource方法\|在FilterSecurityInterceptor中的实现]]
### finallyInvocation方法
复原[[Spring/Spring_Security_04#beforeInvocation方法\|beforeInvocation]]方法中RunAsManager改变的Authentication

### afterInvocation方法
```Java
protected Object afterInvocation(InterceptorStatusToken token, Object returnedObject) {  
    if (token == null) { 
	    //直接返回。在其实现类中可能会走这个分支
       // public object  
       return returnedObject;  
    }    
    //再次调用finallyInvocation
    //原因：有可能我们自定义实现的时候不会走invoke方法中的finally，但是Spring要保证一定会调用finallyInvocation方法来复原可能在beforeInvocation被RunAsManager改变的SecurityContext，防止安全隐患
    finallyInvocation(token); // continue to clean in this method for passivity  
    //此时afterInvocationManager为空
    //在注解实现权限的时候会用到，过滤器实现的时候为null
    if (this.afterInvocationManager != null) {  
       // Attempt after invocation handling  
       try {  
          returnedObject = this.afterInvocationManager.decide(token.getSecurityContext().getAuthentication(),  
                token.getSecureObject(), token.getAttributes(), returnedObject);  
       }       catch (AccessDeniedException ex) {  
          publishEvent(new AuthorizationFailureEvent(token.getSecureObject(), token.getAttributes(),  
                token.getSecurityContext().getAuthentication(), ex));  
          throw ex;  
       }    }    return returnedObject;  
}
```



## FilterInvocation

封装了三个属性
```Java
private FilterChain chain;  
  
private HttpServletRequest request;  
  
private HttpServletResponse response;
```


## ConfigAttribute

必须是一个String，且可以表明所要表达的权限的意思（就是一个权限信息）
可以被RunAsManager, AccessDecisionManager or AccessDecisionManager delegate使用
```Java
public interface ConfigAttribute extends Serializable {  
    String getAttribute();  
}
```

实现类
![Pasted image 20231125231350.png|undefined](/img/user/Pasted%20image%2020231125231350.png)


## FilterSecurityInterceptor授权逻辑

FilterSecurityInterceptor的invoke方法
-> AbstractSecurityInterceptor的beforeInvocation方法
-> AbstractSecurityInterceptor的attemptAuthorization方法
-> AccessDecisionManager的decide方法


## UrlAuthorizationConfigurer

该类并没有实现init和configure方法
### getDecisionVoters方法
```Java
List<AccessDecisionVoter<?>> getDecisionVoters(H http) {  
    List<AccessDecisionVoter<?>> decisionVoters = new ArrayList<>();  
    //根据角色投票
    decisionVoters.add(new RoleVoter());  
    //根据有没有认证去投票
    decisionVoters.add(new AuthenticatedVoter());  
    return decisionVoters;  
}
```
#### RoleVoter
```Java
@Override  
public boolean supports(ConfigAttribute attribute) {  
	//不为空且以_ROLE开头
    return (attribute.getAttribute() != null) && attribute.getAttribute().startsWith(getRolePrefix());  
}  
  
@Override  
public boolean supports(Class<?> clazz) {  
    return true;  
}  
  
@Override  
public int vote(Authentication authentication, Object object, Collection<ConfigAttribute> attributes) {  
    if (authentication == null) {  
       return ACCESS_DENIED;  
    }    
    int result = ACCESS_ABSTAIN;  
    Collection<? extends GrantedAuthority> authorities = extractAuthorities(authentication);  
    for (ConfigAttribute attribute : attributes) {  
	    //support方法判断支不支持
       if (this.supports(attribute)) {  
          result = ACCESS_DENIED;  
          // Attempt to find a matching granted authority  
          for (GrantedAuthority authority : authorities) {  
	          //有一个属性相同返回同意
             if (attribute.getAttribute().equals(authority.getAuthority())) {  
                return ACCESS_GRANTED;  
             }          
		  }       
       }    
    }    
    return result;  
}
```

### createMetadataSource方法
```Java
FilterInvocationSecurityMetadataSource createMetadataSource(H http) {  
    return new DefaultFilterInvocationSecurityMetadataSource(this.registry.createRequestMap());  
}
```

#### 如何构建DefaultFilterInvocationSecurityMetadataSource类中的requestMap属性
通过传入this.registry.createRequestMap()
`private final StandardInterceptUrlRegistry registry;`
##### StandardInterceptUrlRegistry类
为一个注册器，注册请求和请求需要的安全配置的映射的注册器
![Pasted image 20231126173126.png|undefined](/img/user/Pasted%20image%2020231126173126.png)


#### 其中的DefaultFilterInvocationSecurityMetadataSource类

维护了属性requestMap，key为请求匹配器，value为配置属性
object默认为[[Spring/Spring_Security_04#FilterInvocation\|FilterInvocation]]
```Java
public class DefaultFilterInvocationSecurityMetadataSource implements FilterInvocationSecurityMetadataSource {  
  
    protected final Log logger = LogFactory.getLog(getClass());  
  
    private final Map<RequestMatcher, Collection<ConfigAttribute>> requestMap;  
  
    public DefaultFilterInvocationSecurityMetadataSource(  
          LinkedHashMap<RequestMatcher, Collection<ConfigAttribute>> requestMap) {  
       this.requestMap = requestMap;  
    }  
    @Override  
    public Collection<ConfigAttribute> getAllConfigAttributes() {  
       Set<ConfigAttribute> allAttributes = new HashSet<>();  
       this.requestMap.values().forEach(allAttributes::addAll);  
       return allAttributes;  
    }  
    @Override  
    public Collection<ConfigAttribute> getAttributes(Object object) { 
	    //本类中object默认为 FilterInvocation
       final HttpServletRequest request = ((FilterInvocation) object).getRequest();  
       int count = 0;  
       for (Map.Entry<RequestMatcher, Collection<ConfigAttribute>> entry : this.requestMap.entrySet()) {
	       //判断能不能匹配请求  
          if (entry.getKey().matches(request)) {  
             return entry.getValue();  
          }          else {  
             if (this.logger.isTraceEnabled()) {  
                this.logger.trace(LogMessage.format("Did not match request to %s - %s (%d/%d)", entry.getKey(),  
                      entry.getValue(), ++count, this.requestMap.size()));  
             }          
          }       
        }       
        return null;  
    }  
    @Override  
    public boolean supports(Class<?> clazz) {  
       return FilterInvocation.class.isAssignableFrom(clazz);  
    }  
}
```
### 父类AbstractInterceptUrlConfigurer
#### configure方法
```Java
public void configure(H http) throws Exception {  
	//保存对请求的权限处理的配置
    FilterInvocationSecurityMetadataSource metadataSource = createMetadataSource(http);  
    if (metadataSource == null) {  
       return;  
    }    
    //创建一个FilterSecurityInterceptor
    FilterSecurityInterceptor securityInterceptor = createFilterSecurityInterceptor(http, metadataSource,  
          http.getSharedObject(AuthenticationManager.class));  
    if (this.filterSecurityInterceptorOncePerRequest != null) {  
       securityInterceptor.setObserveOncePerRequest(this.filterSecurityInterceptorOncePerRequest);  
    }    securityInterceptor = postProcess(securityInterceptor);  
    http.addFilter(securityInterceptor);  
    http.setSharedObject(FilterSecurityInterceptor.class, securityInterceptor);  
}
```

>在[[Spring/Spring_Security_04#createFilterSecurityInterceptor方法\|createFilterSecurityInterceptor]]方法中构建FilterSecurityInterceptor

>在createMetadataSource中构建FilterInvocationSecurityMetadataSource
>本类中有抽象方法`abstract FilterInvocationSecurityMetadataSource createMetadataSource(H http);`
>[[Spring/Spring_Security_04#createMetadataSource方法\|在UrlAuthorizationConfigurer中的实现]]


#### createFilterSecurityInterceptor方法
```Java
private FilterSecurityInterceptor createFilterSecurityInterceptor(H http,  
       FilterInvocationSecurityMetadataSource metadataSource, AuthenticationManager authenticationManager)  
       throws Exception {
    //直接new  
    FilterSecurityInterceptor securityInterceptor = new FilterSecurityInterceptor();  
    securityInterceptor.setSecurityMetadataSource(metadataSource); 
    //调用getAccessDecisionManager 
    securityInterceptor.setAccessDecisionManager(getAccessDecisionManager(http));  
    securityInterceptor.setAuthenticationManager(authenticationManager);  
    securityInterceptor.setSecurityContextHolderStrategy(getSecurityContextHolderStrategy());  
    securityInterceptor.afterPropertiesSet();  
    return securityInterceptor;  
}
```

#### getAccessDecisionManager方法
```Java
private AccessDecisionManager getAccessDecisionManager(H http) {  
    if (this.accessDecisionManager == null) { 
	    //调用 createDefaultAccessDecisionManager方法
       this.accessDecisionManager = createDefaultAccessDecisionManager(http);  
    }    return this.accessDecisionManager;  
}
```

#### createDefaultAccessDecisionManager方法
```Java
private AccessDecisionManager createDefaultAccessDecisionManager(H http) {  
    AffirmativeBased result = new AffirmativeBased(getDecisionVoters(http));  
    return postProcess(result);  
}
```

##### 关于[[Spring/Spring_Security_04#实现类AffirmativeBased\|AffirmativeBased]]
##### 关于[[Spring/Spring_Security_04#AccessDecisionManager\|AccessDecisionManager]]

##### getDecisionVoters抽象方法

通过getDecisionVoters方法获取投票器
`abstract List<AccessDecisionVoter<?>> getDecisionVoters(H http);`

实现类
![Pasted image 20231126170357.png|undefined](/img/user/Pasted%20image%2020231126170357.png)
[[Spring/Spring_Security_04#getDecisionVoters方法\|在UrlAuthorizationConfigurer中的实现]]
## AccessDecisionManager

判断有没有权限去访问资源

### 抽象实现类AbstractAccessDecisionManager

`private boolean allowIfAllAbstainDecisions = false;`
如果都弃权，能不能访问取决于该属性的值

定义了一堆的`private List<AccessDecisionVoter<?>> decisionVoters;`
将判断具体实现委派给voter
```Java
public interface AccessDecisionVoter<S> {  

	//三种结果，同意，弃权，拒绝
    int ACCESS_GRANTED = 1;  
  
    int ACCESS_ABSTAIN = 0;  
  
    int ACCESS_DENIED = -1;  
  
    boolean supports(ConfigAttribute attribute);  
  
    boolean supports(Class<?> clazz);  
    //通过投票的方式来决定拥有传入的authentication和attributes能不能对object资源进行访问
    int vote(Authentication authentication, S object, Collection<ConfigAttribute> attributes);  
  
}
```

实现类：
![Pasted image 20231126164634.png|undefined](/img/user/Pasted%20image%2020231126164634.png)
分别实现了三种不同的投票规则：一票通过决定，多数通过决定，全部通过决定
#### 实现类AffirmativeBased

decide方法
```Java
public void decide(Authentication authentication, Object object, Collection<ConfigAttribute> configAttributes)  
       throws AccessDeniedException {  
    int deny = 0;  
    for (AccessDecisionVoter voter : getDecisionVoters()) {  
       int result = voter.vote(authentication, object, configAttributes);  
       switch (result) {  
          case AccessDecisionVoter.ACCESS_GRANTED:  
             return;  
          case AccessDecisionVoter.ACCESS_DENIED:  
             deny++;  
             break;  
          default:  
             break;  
       }    
    }    
    //有一个拒绝就抛出异常
    if (deny > 0) {  
       throw new AccessDeniedException(  
             this.messages.getMessage("AbstractAccessDecisionManager.accessDenied", "Access is denied"));  
    }    
    // To get this far, every AccessDecisionVoter abstained  
    //如果弃权，就取决于类中的allowIfAllAbstainDecisions属性
    checkAllowIfAllAbstainDecisions();  
}
```










使用Redis持久化时，登录一次Redis中出现两个token的原因：[[Spring/Spring_Security_02#Redis中出现两个token的原因\|原因]]