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

beforeInvocation(filterInvocation)
## FilterInvocation

封装了三个属性
```Java
private FilterChain chain;  
  
private HttpServletRequest request;  
  
private HttpServletResponse response;
```