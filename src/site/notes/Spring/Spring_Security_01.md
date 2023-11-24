---
{"dg-publish":true,"permalink":"/spring/spring-security-01/","dgPassFrontmatter":true}
---

# SpringSecurity的顶层脉络以及几个核心的类

- ﻿﻿SecurityBuilder
- ﻿﻿HttpSecurity
- ﻿﻿WebSecurity
- ﻿﻿SecurityFilterChain
- ﻿﻿FilterChainProxy

## 引入

WebSecurityConfigurerAdapter弃用，通过配置bean来配置HttpSecurity和WebSecurity
```
Use a org.springframework.security.web.SecurityFilterChain Bean to configure HttpSecurity or a WebSecurityCustomizer Bean to configure WebSecurity
```


HttpSecurity和WebSecurity都实现了SecurityBuilder，用来构建对象

```
public interface SecurityBuilder<O> {
	O build() throws Exception;
}
```

## HttpSecurity

![Pasted image 20231105144050.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231105144050.png)

```
public final class HttpSecurity 
	extends AbstractConfiguredSecurityBuilder<DefaultSecurityFilterChain, HttpSecurity>
	implements SecurityBuilder<DefaultSecurityFilterChain>,HttpSecurityBuilder<HttpSecurity>{}
```

>implements SecurityBuilder\<DefaultSecurityFilterChain>：
>HttpSecurity的泛型O为DefaultSecurityFilterChain，即为了构建DefaultSecurityFilterChain

### SecurityFilterChain


```
public interface SecurityFilterChain {  
	//如果返回true，就会对request应用list里的过滤器
    boolean matches(HttpServletRequest request);  
  
    List<Filter> getFilters();  
  
}
```

#### 实现类DefaultSecurityFilterChain
![Pasted image 20231105145428.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231105145428.png)
```Java
public final class DefaultSecurityFilterChain implements SecurityFilterChain {  
  
    private static final Log logger = LogFactory.getLog(DefaultSecurityFilterChain.class);  
  
    private final RequestMatcher requestMatcher;  
  
    private final List<Filter> filters;  
  
    public DefaultSecurityFilterChain(RequestMatcher requestMatcher, Filter... filters) {  
       this(requestMatcher, Arrays.asList(filters));  
    }  
  
    public DefaultSecurityFilterChain(RequestMatcher requestMatcher, List<Filter> filters) {  
       if (filters.isEmpty()) {  
          logger.info(LogMessage.format("Will not secure %s", requestMatcher));  
       }  
       else {  
          logger.info(LogMessage.format("Will secure %s with %s", requestMatcher, filters));  
       }  
       this.requestMatcher = requestMatcher;  
       this.filters = new ArrayList<>(filters);  
    }  
  
    public RequestMatcher getRequestMatcher() {  
       return this.requestMatcher;  
    }  
  
    @Override  
    public List<Filter> getFilters() {  
       return this.filters;  
    }  
  
    @Override  
    public boolean matches(HttpServletRequest request) {  //匹配成功就会应用filters里的过滤器
       return this.requestMatcher.matches(request);  
    }  
  
    @Override  
    public String toString() {  
       return this.getClass().getSimpleName() + " [RequestMatcher=" + this.requestMatcher + ", Filters=" + this.filters  
             + "]";  
    }  
  
}
```

## WebSecurity

![Pasted image 20231105144145.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231105144145.png)

```
public final class WebSecurity 
	extends AbstractConfiguredSecurityBuilder<Filter, WebSecurity> 
	implements SecurityBuilder<Filter>, ApplicationContextAware, ServletContextAware {}
```
implements SecurityBuilder\<Filter>:WebSecurity的泛型O为Filter，即为了构建Filter

### SecurityBuilder的build方法 -> doBuild方法 

```
protected final O doBuild() throws Exception {  
    synchronized (this.configurers) {  
       this.buildState = BuildState.INITIALIZING;  
       beforeInit();  
       init();  
       this.buildState = BuildState.CONFIGURING;  
       beforeConfigure();  
       configure();  
       this.buildState = BuildState.BUILDING;  
       O result = performBuild();  
       this.buildState = BuildState.BUILT;  
       return result;  
    }  
}
```
其中有些方法不是abstract，说明这些方法不重要，子类不需要重写（abstract的方法必须实现）
由代码可见通过**performBuild**方法构建了result对象
performBuild方法有三个实现类，其中包括熟悉的HttpSecurity和WebSecurity 

>在HttpSecurity中，该方法 return new DefaultSecurityFilterChain(this.requestMatcher, sortedFilters); 

>在WebSecurity中，该方法返回了一个 [[Spring/Spring_Security_01#filterChainProxy\|FilterChainProxy]]
```Java
protected Filter performBuild() throws Exception {  
    Assert.state(!this.securityFilterChainBuilders.isEmpty(),  
          () -> "At least one SecurityBuilder<? extends SecurityFilterChain> needs to be specified. "  
                + "Typically this is done by exposing a SecurityFilterChain bean. "  
                + "More advanced users can invoke " + WebSecurity.class.getSimpleName()  
                + ".addSecurityFilterChainBuilder directly");  
    int chainSize = this.ignoredRequests.size() + this.securityFilterChainBuilders.size();  
    List<SecurityFilterChain> securityFilterChains = new ArrayList<>(chainSize);  
    List<RequestMatcherEntry<List<WebInvocationPrivilegeEvaluator>>> requestMatcherPrivilegeEvaluatorsEntries = new ArrayList<>();  
    for (RequestMatcher ignoredRequest : this.ignoredRequests) {  
       WebSecurity.this.logger.warn("You are asking Spring Security to ignore " + ignoredRequest  
             + ". This is not recommended -- please use permitAll via HttpSecurity#authorizeHttpRequests instead.");  
       SecurityFilterChain securityFilterChain = new DefaultSecurityFilterChain(ignoredRequest);  
       securityFilterChains.add(securityFilterChain);  
       requestMatcherPrivilegeEvaluatorsEntries  
          .add(getRequestMatcherPrivilegeEvaluatorsEntry(securityFilterChain));  
    }  
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


WebSecurity的build方法由[[Spring/Spring_Security_02#WebSecurityConfiguration\|WebSecurityConfiguration]]调用
### FilterChainProxy

![Pasted image 20231105185837.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231105185837.png)

其构造方法传入了[[Spring/Spring_Security_01#SecurityFilterChain\|SecurityFilterChain]]的集合
```
public FilterChainProxy(List<SecurityFilterChain> filterChains) {  
    this.filterChains = filterChains;  
}
```

其中doFilter方法 -> doFilterInternal方法 
```Java
@Override  
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)  
       throws IOException, ServletException {  
    boolean clearContext = request.getAttribute(FILTER_APPLIED) == null;  
    if (!clearContext) {  
       doFilterInternal(request, response, chain);  
       return;  
    }  
    try {  
       request.setAttribute(FILTER_APPLIED, Boolean.TRUE);  
       doFilterInternal(request, response, chain);  
    }  
    catch (Exception ex) {  
       Throwable[] causeChain = this.throwableAnalyzer.determineCauseChain(ex);  
       Throwable requestRejectedException = this.throwableAnalyzer  
          .getFirstThrowableOfType(RequestRejectedException.class, causeChain);  
       if (!(requestRejectedException instanceof RequestRejectedException)) {  
          throw ex;  
       }  
       this.requestRejectedHandler.handle((HttpServletRequest) request, (HttpServletResponse) response,  
             (RequestRejectedException) requestRejectedException);  
    }  
    finally {  
       this.securityContextHolderStrategy.clearContext();  
       request.removeAttribute(FILTER_APPLIED);  
    }  
}

private void doFilterInternal(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {  //FilterChain：最顶层的filter，例如tomcat中的
    FirewalledRequest firewallRequest = this.firewall.getFirewalledRequest((HttpServletRequest) request);  
    HttpServletResponse firewallResponse = this.firewall.getFirewalledResponse((HttpServletResponse) response); 
    //得到filters
    List<Filter> filters = 1️⃣getFilters(firewallRequest);  
    if (filters == null || filters.size() == 0) {  
       if (logger.isTraceEnabled()) {  
          logger.trace(LogMessage.of(() -> "No security for " + requestLine(firewallRequest)));  
       }  
       firewallRequest.reset();  

       this.2️⃣filterChainDecorator.decorate(chain).doFilter(firewallRequest, firewallResponse);  
       return;  
    }  
    if (logger.isDebugEnabled()) {  
       logger.debug(LogMessage.of(() -> "Securing " + requestLine(firewallRequest)));  
    }  
    FilterChain reset = (req, res) -> {  
       if (logger.isDebugEnabled()) {  
          logger.debug(LogMessage.of(() -> "Secured " + requestLine(firewallRequest)));  
       }  
       // Deactivate path stripping as we exit the security filter chain  
       firewallRequest.reset();  
       chain.doFilter(req, res);  
    };  
    this.filterChainDecorator.decorate(reset, filters).doFilter(firewallRequest, firewallResponse);  //调用doFilter
} 
```

>该方法使filters(List\<Filter> filters = getFilters(firewallRequest);  )中的过滤器得到应用
>filters通过filterChainProxy构造方法传入

1️⃣getFilters(HttpServletRequest request)
```
private List<Filter> getFilters(HttpServletRequest request) {  
    int count = 0;  
    for (SecurityFilterChain chain : this.filterChains) {  
       if (logger.isTraceEnabled()) {  
          logger.trace(LogMessage.format("Trying to match request against %s (%d/%d)", chain, ++count,  
                this.filterChains.size()));  
       }  

		//🌟只会执行filterChains中第一个匹配的SecurityFilterChain
       if (chain.matches(request)) {  
          return chain.getFilters();  //得到SecurityFilterChain中所有的过滤器
       }  
    }  
    return null;  
}
```

2️⃣关于spring6.1中的VirtualFilterChain：
```
private FilterChainProxy.FilterChainDecorator filterChainDecorator
    = new FilterChainProxy.VirtualFilterChainDecorator()
```

## SpringSecurity流程

>一个FilterChainProxy，其中有一堆SecurityFilterChain，每个传过来的请求都要经过requestMatcher来匹配，如果匹配就应用SecurityFilterChain其中的一堆过滤器。
>程序一般只有一个SecurityFilterChain。
>FilterChainProxy调用doFilter方法，再调用doFilterInternal方法，通过getFilters(HttpServletRequest request)方法通过传入的request得到第一个匹配的SecurityFilterChain，通过getFilters()方法得到SecurityFilterChain中所有的过滤器，构建为一个VirtualFilterChain，最后调用每个过滤器的doFilter方法。


## HttpSecurity和WebSecurity的配置

```
public class SecurityConfig {  
    @Bean  
    public DefaultSecurityFilterChain securityFilterChain(HttpSecurity http) { 
	    ...... 
        return http.build();  
    }  
  
    @Bean  
    public WebSecurityCustomizer webSecurityCustomizer() {  
        return web -> {  
            ......
        }  
    }  
}
```


---

# 三个核心类

- ﻿﻿HttpSecurity 
- ﻿﻿SecurityConfigurer
- AbstractConfiguredSecurityBuilder(间接实现了SecurityBuilder)

>SecurityBuilder用来构建安全对象，安全对象包括：HttpSecurity、FilterChainProxy、AuthenticationManager SecurityConfigurer用来配置安全对象构建器（SecurityBuilder），典型的有：FormLoginConfigurer、CsrfConfigurer

## SecurityConfigurer

>安全构建器SecurityBuilder的配置器

>SecurityBuilder用来构建一个对象，SecurityConfigurer用来配置一个SecurityBuilder
```Java
/**  
 * Allows for configuring a {@link SecurityBuilder}. All {@link SecurityConfigurer} first  
 * have their {@link #init(SecurityBuilder)} method invoked. After all  
 * {@link #init(SecurityBuilder)} methods have been invoked, each  
 * {@link #configure(SecurityBuilder)} method is invoked.  
 * //必须先调用init再调用configure
 * * @param <O> The object being built by the {@link SecurityBuilder} B  
 * @param <B> The {@link SecurityBuilder} that builds objects of type O. This is also the  
 * {@link SecurityBuilder} that is being configured.  
 * @author Rob Winch * @see AbstractConfiguredSecurityBuilder  
 */  
public interface SecurityConfigurer<O, B extends SecurityBuilder<O>> {  
  
    /**  
     * Initialize the {@link SecurityBuilder}. Here only shared state should be created  
     * and modified, but not properties on the {@link SecurityBuilder} used for building  
     * the object. This ensures that the {@link #configure(SecurityBuilder)} method uses  
     * the correct shared objects when building. Configurers should be applied here.     * @param builder  
     * @throws Exception  
     */  
    void init(B builder) throws Exception;  
  
    /**  
     * Configure the {@link SecurityBuilder} by setting the necessary properties on the  
     * {@link SecurityBuilder}.  
     * @param builder  
     * @throws Exception  
     */  
    void configure(B builder) throws Exception;  
  
}
```


## ﻿﻿AbstractConfiguredSecurityBuilder

![Pasted image 20231105202058.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231105202058.png)

>类图中AbstractSecurityBuilder的作用：控制build只能构建一次

**AbstractConfiguredSecurityBuilder被SecurityConfigurer所配置**

AbstractConfiguredSecurityBuilder类：
```Java
public abstract class AbstractConfiguredSecurityBuilder<O, B extends SecurityBuilder<O>>  
       extends AbstractSecurityBuilder<O> {  
  
    private final Log logger = LogFactory.getLog(getClass());  
	//被一堆的SecurityConfigurer所配置
    private final LinkedHashMap<Class<? extends SecurityConfigurer<O, B>>, List<SecurityConfigurer<O, B>>> configurers = new LinkedHashMap<>();  
  
    private final List<SecurityConfigurer<O, B>> configurersAddedInInitializing = new ArrayList<>();  
  
    private final Map<Class<?>, Object> sharedObjects = new HashMap<>();  
	
    private final boolean allowConfigurersOfSameType;  
	//构建状态
    private BuildState buildState = BuildState.UNBUILT;  
	//对象后置处理
	//可以实现里面的postProcesser方法自定义处理
    private ObjectPostProcessor<Object> objectPostProcessor;

	......

	public <C> void setSharedObject(Class<C> sharedType, C object) {  
	    this.sharedObjects.put(sharedType, object);  
	}   
	public <C> C getSharedObject(Class<C> sharedType) {  
	    return (C) this.sharedObjects.get(sharedType);  
	}

	......
	//🌟add方法
	private <C extends SecurityConfigurer<O, B>> void add(C configurer) {  
	    Assert.notNull(configurer, "configurer cannot be null");  
	    Class<? extends SecurityConfigurer<O, B>> clazz = (Class<? extends SecurityConfigurer<O, B>>) configurer  
	       .getClass();  
	    synchronized (this.configurers) {  
	       if (this.1️⃣buildState.isConfigured()) {  //判断是否已经configured，已经构建的不能应用配置器
	          throw new IllegalStateException("Cannot apply " + configurer + " to already built object");  
	       }  
	       List<SecurityConfigurer<O, B>> configs = null;  
	       if (this.allowConfigurersOfSameType) {  //是否允许同一个类型的configurer
	          configs = this.configurers.get(clazz);  
	       }  
	       //如果不允许，则configs一定为null
	       configs = (configs != null) ? configs : new ArrayList<>(1);  //如果==null，先初始化
	       configs.add(configurer);  
	       this.configurers.put(clazz, configs);  
	       3️⃣🌟//初始化过程中加入的configure加入configurersAddedInInitializing
	       if (this.buildState.isInitializing()) {  
	          this.configurersAddedInInitializing.add(configurer);  
	       }  
	    }  
	}
	......

	@Override  
	protected final O doBuild() throws Exception {  
	    synchronized (this.configurers) {  
	       this.buildState = BuildState.INITIALIZING;  //状态变为正在初始化
	       beforeInit();  
	       init();  4️⃣//初始化
	       this.buildState = BuildState.CONFIGURING; //状态改为正在配置 
	       beforeConfigure();5️⃣
	       configure();6️⃣
	       this.buildState = BuildState.BUILDING;  //状态改为正在构建
	       //🌟7️⃣performBuild()是一个抽象方法，必须子类实现，真正实现构建的方法
	       O result = performBuild();   
	       this.buildState = BuildState.BUILT;  
	       return result;  
	    }  
	}


	......
	4️⃣
	private void init() throws Exception {  
	    Collection<SecurityConfigurer<O, B>> configurers = 2️⃣getConfigurers();  
	    //所有的配置器每个初始化 
	    for (SecurityConfigurer<O, B> configurer : configurers) {  
	       configurer.init((B) this);  
	    }  
	    for (SecurityConfigurer<O, B> configurer : this.configurersAddedInInitializing) {  
	       configurer.init((B) this);  
	    }  
	} 


	2️⃣ 
	private Collection<SecurityConfigurer<O, B>> getConfigurers() {  
	    List<SecurityConfigurer<O, B>> result = new ArrayList<>();  
	    for (List<SecurityConfigurer<O, B>> configs : this.configurers.values()) {  
	       result.addAll(configs);  
	    }  
	    return result;  
	}

	.......
	
	6️⃣//congigure方法
	private void configure() throws Exception {  
	    Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();  
	    for (SecurityConfigurer<O, B> configurer : configurers) {  
	       configurer.configure((B) this);  
	    }  
	}  
	  
	private Collection<SecurityConfigurer<O, B>> getConfigurers() {  
	    List<SecurityConfigurer<O, B>> result = new ArrayList<>();  
	    for (List<SecurityConfigurer<O, B>> configs : this.configurers.values()) {  
	       result.addAll(configs);  
	    }  
	    return result;  
	}
}
```


##### 4️⃣init()
###### Collection<SecurityConfigurer<O, B>> configurers = 2️⃣getConfigurers()作用：
>此时再对configurers执行put，集合中不会再添加新的元素了，因为getConfigurers方法创建了一个新的集合并返回。
###### for (SecurityConfigurer<O, B> configurer : this.configurersAddedInInitializing)作用：
>执行第一个for的时候，configurers中的元素执行init方法时候可能会执行add一个configure，而这个时候add的configure不能加入configures集合了，只能加入最初的configures map。
>但是这些configure也要执行init方法，因此建立属性private final List<SecurityConfigurer<O, B>> configurersAddedInInitializing = new ArrayList<>();，将这些configure加入configurersAddedInInitializing（3️⃣），遍历执行init方法。
>**这就是为什么要建立configurersAddedInInitializing属性，以及add方法中3️⃣处的作用。**

>问题：configurersAddedInInitializing中的configure执行init方法的时候再执行add一个configure会发生什么？

>验证：在groovy console中写入
```
List<String> list = new ArrayList<>()
list.add ("1")
list.add ("2")
for(String item in list) {
	if (item.equals ("1")) {
	list. add ("3")
	}
	System.out.println(item)
｝
```
>执行抛出异常java.util.ConcurrentModificationException
>因为执行add的时候ArrayList的modcount++，不等于expectedModCount，会报错

##### 5️⃣beforeConfigure()
在本类中为空方法，在HttpSecurity中有一个实现
```
@Override  
protected void beforeConfigure() throws Exception {  
    if (this.authenticationManager != null) {  
       setSharedObject(AuthenticationManager.class, this.authenticationManager);  
    }  
    else {  
       ObservationRegistry registry = getObservationRegistry();  
       AuthenticationManager manager = getAuthenticationRegistry().build();  
       if (!registry.isNoop() && manager != null) {  
          setSharedObject(AuthenticationManager.class, new ObservationAuthenticationManager(registry, manager));  
       }  
       else {  
          setSharedObject(AuthenticationManager.class, manager);  
       }  
    }  
}
```

##### 1️⃣BuildState
```
private enum BuildState {  
  
    /**  
     * This is the state before the {@link Builder#build()} is invoked     */    UNBUILT(0),  
  
    /**  
     * The state from when {@link Builder#build()} is first invoked until all the     * {@link SecurityConfigurer#init(SecurityBuilder)} methods have been invoked.  
     */    INITIALIZING(1),  
  
    /**  
     * The state from after all {@link SecurityConfigurer#init(SecurityBuilder)} have  
     * been invoked until after all the     * {@link SecurityConfigurer#configure(SecurityBuilder)} methods have been  
     * invoked.     */    CONFIGURING(2),  
  
    /**  
     * From the point after all the     * {@link SecurityConfigurer#configure(SecurityBuilder)} have completed to just  
     * after {@link AbstractConfiguredSecurityBuilder#performBuild()}.  
     */    BUILDING(3),  
  
    /**  
     * After the object has been completely built.     */    BUILT(4);  
  
    private final int order;  
  
    BuildState(int order) {  
       this.order = order;  
    }  
  
    public boolean isInitializing() {  
       return INITIALIZING.order == this.order;  
    }  
  
    /**  
     * Determines if the state is CONFIGURING or later     * @return     */    public boolean isConfigured() {  
       return this.order >= CONFIGURING.order;  
    }  
  
}
```

## HttpSecurity

##### 7️⃣HttpSecurity对performBuild()方法的实现
```Java
@Override  
protected DefaultSecurityFilterChain performBuild() {  
    ExpressionUrlAuthorizationConfigurer<?> expressionConfigurer = getConfigurer(  
          ExpressionUrlAuthorizationConfigurer.class);  
    AuthorizeHttpRequestsConfigurer<?> httpConfigurer = getConfigurer(AuthorizeHttpRequestsConfigurer.class);  
    boolean oneConfigurerPresent = expressionConfigurer == null ^ httpConfigurer == null; 

	 //该判断的含义：expressionConfigurer和httpConfigurer只能配置一个
    Assert.state((expressionConfigurer == null && httpConfigurer == null) || oneConfigurerPresent,  
          "authorizeHttpRequests cannot be used in conjunction with authorizeRequests. Please select just one.");  
    this.filters.sort(OrderComparator.INSTANCE);  
    List<Filter> sortedFilters = new ArrayList<>(this.filters.size());  
    for (Filter filter : this.filters) {  
       sortedFilters.add(((OrderedFilter) filter).filter);  
    }  
    return new DefaultSecurityFilterChain(this.requestMatcher, sortedFilters);  
}
```

>该方法最终构建出DefaultSecurityFilterChain（属性为一堆filters和一个requestMatcher）

>在HttpSecurity中，默认的requestMatcher为：
```
private RequestMatcher requestMatcher = AnyRequestMatcher.INSTANCE;
```
>其matches方法返回true，匹配所有request
>
>在HttpSecurity的requestMatcher方法中，可以自定义requestMatcher
```
public HttpSecurity requestMatcher (RequestMatcher requestMatcher) {
	this.requestMatcher = requestMatcher;
	return this;
｝
```

---

# 四个核心类

- ﻿﻿AuthenticationManager  
	一个安全对象
	用来认证，认证的时候需要一个Authentication
	认证后返回一个标记为已认证的Authentication对象

- ﻿﻿AuthenticationManagerBuilder  
	用来构建AuthenticationManager的SecurityBuilder
	继承自AbstractConfiguredSecurityBuilder

- ﻿﻿ProviderManager
	AuthenticationManager的具体实现
	一个ProviderManager中有很多AuthenticationProvider

- AuthenticationProvider
	认证策略模式的实现类

## Authentication

只是封装认证信息。没有任何逻辑。
```
public interface Authentication extends Principal, Serializable {  
	//获取权限/角色
	Collection<? extends GrantedAuthority> getAuthorities();  
	//获取凭证（如密码）
	Object getCredentials(); 
	//获取详细信息 
	Object getDetails();  
	//获取主体（如用户名）
	Object getPrincipal();  
	boolean isAuthenticated();  
	void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;  
}
```

有一个重要的实现类UsernamePasswordAuthenticationToken

## AuthenticationManager

```
public interface AuthenticationManager {  
	Authentication authenticate(Authentication authentication) throws AuthenticationException;  
}
```

authenticate方法返回一个填充好所有属性的Authentication对象


## AuthenticationManagerBuilder

描述：
```
SecurityBuilder used to create an AuthenticationManager. Allows for easily building in memory authentication, LDAP authentication, JDBC based authentication, adding UserDetailsService, and adding AuthenticationProvider's.
```
![Pasted image 20231106215901.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231106215901.png)
### 属性
```
public class AuthenticationManagerBuilder  
       extends AbstractConfiguredSecurityBuilder<AuthenticationManager, AuthenticationManagerBuilder>  
       implements ProviderManagerBuilder<AuthenticationManagerBuilder> {  
  
    private final Log logger = LogFactory.getLog(getClass());  
  
    private AuthenticationManager parentAuthenticationManager;  
  
    private List<AuthenticationProvider> authenticationProviders = new ArrayList<>();  
  
    private UserDetailsService defaultUserDetailsService;  
  
    private Boolean eraseCredentials;  
  
    private AuthenticationEventPublisher eventPublisher;

	......
```

继承自AbstractConfiguredSecurityBuilder，而这个抽象类最重要的方法是**performBuild方法**

### performBuild()

```
@Override  
protected ProviderManager performBuild() throws Exception {  
    if (!isConfigured()) {  //如果没有配置就想构建，返回null
       this.logger.debug("No authenticationProviders and no parentAuthenticationManager defined. Returning null.");  
       return null;  
    }  
    ProviderManager providerManager = new ProviderManager(this.authenticationProviders, this.parentAuthenticationManager);  
    if (this.eraseCredentials != null) { //清除凭证 
       providerManager.setEraseCredentialsAfterAuthentication(this.eraseCredentials);  
    }  
    if (this.eventPublisher != null) {  //设置时间发布器
       providerManager.setAuthenticationEventPublisher(this.eventPublisher);  
    }  
    providerManager = postProcess(providerManager);  //后处理
    return providerManager;  
}
```

>ProviderManager providerManager = new ProviderManager(this.authenticationProviders, this.parentAuthenticationManager)：
>设置authenticationProviders，即一堆的[[Spring/Spring_Security_01#AuthenticationProvider\|AuthenticationProvider]]；和parentAuthenticationManager，即父类的[[Spring/Spring_Security_01#AuthenticationManager\|AuthenticationManager]]

>返回值为[[Spring/Spring_Security_01#ProviderManager\|ProviderManager]]
#### postProcess(providerManager)

后处理。可以实现ObjectPostProcesser接口，实现postProcesser方法实现自定义配置
```
public interface ObjectPostProcessor<T> { 
	<O extends T> O postProcess (O object);
｝
```

### 属性设置

#### AuthenticationProvider在这个构造方法被设置
```
@Override  
public AuthenticationManagerBuilder authenticationProvider(AuthenticationProvider authenticationProvider) {  
    this.authenticationProviders.add(authenticationProvider);  
    return this;  
}
```

#### parentAuthenticationManager在这个构造方法被设置
```
public AuthenticationManagerBuilder parentAuthenticationManager(AuthenticationManager authenticationManager) {  
    if (authenticationManager instanceof ProviderManager) {  
       eraseCredentials(((ProviderManager) authenticationManager).isEraseCredentialsAfterAuthentication());  
    }  
    this.parentAuthenticationManager = authenticationManager;  
    return this;  
}
```

## AuthenticationProvider

```
public interface AuthenticationProvider {
	Authentication authenticate(Authentication authentication) throws AuthenticationException;
	//是否支持认证，返回true就调用authenticate方法
	
    boolean supports(Class<?> authentication);  
}
```

#### authenticate(Authentication authentication)

和[[Spring/Spring_Security_01#AuthenticationManager\|AuthenticationManager]]的authenticate方法相同

#### 策略模式

一个问题多种解法。特点有解耦，职责单一，开闭原则
## ProviderManager

![Pasted image 20231106220432.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231106220432.png)

```Java
public class ProviderManager implements AuthenticationManager, MessageSourceAware, InitializingBean {  
  
    private static final Log logger = LogFactory.getLog(ProviderManager.class);  
  
    private AuthenticationEventPublisher eventPublisher = new NullEventPublisher();  
  
    private List<AuthenticationProvider> providers = Collections.emptyList();  1️⃣
  
    protected MessageSourceAccessor messages = SpringSecurityMessageSource.getAccessor();  
  
    private AuthenticationManager parent;  1️⃣
  
    private boolean eraseCredentialsAfterAuthentication = true;


	......

	@Override  
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {  
	    Class<? extends Authentication> toTest = authentication.getClass(); //测试authentication是否支持
	    AuthenticationException lastException = null;  
	    AuthenticationException parentException = null;  
	    Authentication result = null;  
	    Authentication parentResult = null;  
	    int currentPosition = 0;  
	    int size = this.providers.size();  
	    //逐个provider去调用toTest方法
	    for (AuthenticationProvider provider : getProviders()) {  
	       if (!provider.supports(toTest)) {  
	          continue;  
	       }  
	       //如果支持，继续认证
	       if (logger.isTraceEnabled()) {  
	          logger.trace(LogMessage.format("Authenticating request with %s (%d/%d)",  
	                provider.getClass().getSimpleName(), ++currentPosition, size));  
	       }  
	       try {  
	          result = provider.authenticate(authentication);  
	          if (result != null) {  
	             copyDetails(authentication, result);  //将authentication通过copyDetails赋值给result
	             break;  
	          }  
	       }  
	       catch (AccountStatusException | InternalAuthenticationServiceException ex) {  
	          prepareException(ex, authentication);  
	          // SEC-546: Avoid polling additional providers if auth failure is due to  
	          // invalid account status          throw ex;  
	       }  
	       catch (AuthenticationException ex) {  
	          lastException = ex;  
	       }  
	    }  
	    //for循环中的provider都不支持，result == null
	    if (result == null && this.parent != null) {  
	       // Allow the parent to try.  
	       try {  
	          parentResult = this.parent.authenticate(authentication);  //2️⃣调用parent的authenticate，测试parent的provider是否支持
	          result = parentResult;  
	       }  
	       catch (ProviderNotFoundException ex) {  
	          // ignore as we will throw below if no other exception occurred prior to  
	          // calling parent and the parent          // may throw ProviderNotFound even though a provider in the child already          // handled the request       }  
	       catch (AuthenticationException ex) {  
	          parentException = ex;  
	          lastException = ex;  
	       }  
	    }  
	    if (result != null) {  
	       if (this.eraseCredentialsAfterAuthentication && (result instanceof CredentialsContainer)) {  //是否擦除凭证信息。登录成功在内存擦出凭证信息，保护隐私
	          // Authentication is complete. Remove credentials and other secret data  
	          // from authentication          ((CredentialsContainer) result).eraseCredentials();  
	       }  
	       // If the parent AuthenticationManager was attempted and successful then it  
	       // will publish an AuthenticationSuccessEvent       
	       // This check prevents a duplicate AuthenticationSuccessEvent if the parent       
	       // AuthenticationManager already published it       
	       if (3️⃣parentResult == null) {
	          this.eventPublisher.publishAuthenticationSuccess(result);  
	       }  
	  
	       return result;  
	    }  
	  
	    // Parent was null, or didn't authenticate (or throw an exception).  
	    //没有一个provicer可以支持认证
	    if (lastException == null) {  
	       lastException = new ProviderNotFoundException(this.messages.getMessage("ProviderManager.providerNotFound",  
	             new Object[] { toTest.getName() }, "No AuthenticationProvider found for {0}"));  
	    }  
	    // If the parent AuthenticationManager was attempted and failed then it will  
	    // publish an AbstractAuthenticationFailureEvent    // This check prevents a duplicate AbstractAuthenticationFailureEvent if the    // parent AuthenticationManager already published it    if (parentException == null) {  
	       prepareException(lastException, authentication);  
	    }  
	    throw lastException;  
	}
```

### authenticate方法

>1️⃣先使用providers中的AuthenticationProvider，如果都不支持，就调用父类的AuthenticationProvider。
>如果再认证不过，就报错


>2️⃣parent的provider中，最主要的还是ProviderManager
![Pasted image 20231106224754.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231106224754.png)

>3️⃣判断事件是由谁发生的。返回false说明事件由父类完成认证，父类完成的认证父类来发布。同时避免重复发布。
#### 流程
子类的provider逐个认证，如果认证都不成功，就用父类的provider逐个认证。如果子类和父类中没有一个provider支持认证，就抛出异常。
如果认证成功，将authentication通过copyDetails赋值给result


## 总结

**AuthenticationManagerBuilder构建AuthenticationManager，实际上就是构建ProviderManager（是AuthenticationManager的一个具体实现）
AuthenticationProvider是真正认证请求的
ProviderManager维护了一堆的AuthenticationProvider，认证的时候调用AuthenticationProvider的authenticate方法**

---

# 五个核心类

- ﻿﻿UserDetailsService

- ﻿﻿DaoAuthenticationProvider
	数据库相关认证

- ﻿﻿AbstractUserDetailsAuthenticationProvider
	DaoAuthenticationProvider的一个父类

- ﻿﻿UsernamePasswordAuthenticationFilter


- ﻿﻿FormLoginConfigurer
	是一个[[Spring/Spring_Security_01#SecurityConfigurer\|SecurityConfigurer]]
	用来配置UsernamePasswordAuthenticationFilter
	配置的是HttpSecurityBuilder（[[Spring/Spring_Security_01#HttpSecurity\|HttpSecurity]]的构建器）


## UserDetailsService

通过用户名加载用户信息

```
public interface UserDetailsService {  
	UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;  
}
```

实现类：
![Pasted image 20231107150749.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231107150749.png)

## DaoAuthenticationProvider

![Pasted image 20231107151019.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231107151019.png)

AuthenticationProvider的实现类，通过UserDetailsService用来获取用户详情
父类为[[Spring/Spring_Security_01#AbstractUserDetailsAuthenticationProvider\|AbstractUserDetailsAuthenticationProvider]]

## AbstractUserDetailsAuthenticationProvider

**A base AuthenticationProvider that allows subclasses to override and work with UserDetails objects. The class is designed to respond to UsernamePasswordAuthenticationToken authentication requests.**

是[[Spring/Spring_Security_01#AuthenticationProvider\|AuthenticationProvider]]的实现类

### 🌟authenticate方法

```Java
public Authentication authenticate(Authentication authentication) throws AuthenticationException {  
    Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,  
          () -> this.messages.getMessage("AbstractUserDetailsAuthenticationProvider.onlySupports", 
                "Only UsernamePasswordAuthenticationToken is supported")); 

	//拿到username
    String username = 1️⃣determineUsername(authentication);  
    //是否使用缓存
    boolean cacheWasUsed = true;  
    //2️⃣获取缓存
    UserDetails user = this.userCache.getUserFromCache(username);  
    if (user == null) {  
	    //缓存里没有user
	    //事实上user永远等于null
       cacheWasUsed = false;  
       try {  
		    //拿到用户的user对象
		    user = 6️⃣retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);  
       }  
       catch (UsernameNotFoundException ex) {  
          this.logger.debug("Failed to find user '" + username + "'");  
          //3️⃣是否隐藏UserNotFoundExceptions
          if (!this.hideUserNotFoundExceptions) {  
             throw ex;  
          }  
          throw new BadCredentialsException(this.messages  
             .getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));  
       }  
       Assert.notNull(user, "retrieveUser returned null - a violation of the interface contract");  
    }  
    try {  
	    //校验
       this.preAuthenticationChecks.check(user);  
       additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);  
    } 
    //4️⃣ 
    catch (AuthenticationException ex) {  
       if (!cacheWasUsed) {  
          throw ex;  
       }  
       // There was a problem, so try again after checking  
       // we're using latest data (i.e. not from the cache)       cacheWasUsed = false;  
       user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);  
       this.preAuthenticationChecks.check(user);  
       additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);  
    }  
    //检验
    this.postAuthenticationChecks.check(user);  
    if (!cacheWasUsed) {  //如果没用缓存，就加入缓存
       this.userCache.putUserInCache(user);  
    }  
    Object principalToReturn = user;  
    if (this.forcePrincipalAsString) {  //如果强制principal为String
       principalToReturn = user.getUsername();  
    }  
    return 5️⃣createSuccessAuthentication(principalToReturn, authentication, user);  
}
```


#### 1️⃣determineUsername

```
private String determineUsername(Authentication authentication) {  
    return (authentication.getPrincipal() == null) ? "NONE_PROVIDED" : authentication.getName();  
}
```

#### 2️⃣user永远等于null

>this.userCache的赋值为：
```
private UserCache userCache = new NullUserCache();
```

而NullUserCache实现了UserCache接口。
UserCache是为了在远程连接的时候避免频繁调用数据库，于是可以将用户信息存储在UserCache中。
但是一般我们都保存在session中，所以SpringSecurity默认给this.userCache赋值为NullUserCache，不使用UserCache
所以2️⃣位置user永远等于null

也可以实现自己的UserCache，该类中提供了方法setUserCache
```
public void setUserCache(UserCache userCache) {  
    this.userCache = userCache;  
}
```

#### 3️⃣关于隐藏UserNotFoundExceptions

3️⃣处判断是否隐藏UserNotFoundExceptions。隐藏用户名的正确性，是一种安全保护机制


#### 4️⃣如果捕获到了AuthenticationException

>如果不是从缓存获取，则抛异常。

>如果是从缓存获取，有一种可能是缓存中用登录凭证信息更新不及时（与数据库信息不同）导致异常，所以再此调用retrieveUser方法获取user并进行校验


#### 关于校验

该类的属性中：
```
private UserDetailsChecker preAuthenticationChecks = new DefaultPreAuthenticationChecks();  
  
private UserDetailsChecker postAuthenticationChecks = new DefaultPostAuthenticationChecks();
```

##### UserDetailsChecker
```
public interface UserDetailsChecker {  
	void check(UserDetails toCheck);  
}
```

##### DefaultPreAuthenticationChecks
```Java
private class DefaultPreAuthenticationChecks implements UserDetailsChecker {  
  
    @Override  
    public void check(UserDetails user) {  
       if (!user.isAccountNonLocked()) {  
          AbstractUserDetailsAuthenticationProvider.this.logger  
             .debug("Failed to authenticate since user account is locked");  
          throw new LockedException(AbstractUserDetailsAuthenticationProvider.this.messages  
             .getMessage("AbstractUserDetailsAuthenticationProvider.locked", "User account is locked"));  
       }  
       if (!user.isEnabled()) {  
          AbstractUserDetailsAuthenticationProvider.this.logger  
             .debug("Failed to authenticate since user account is disabled");  
          throw new DisabledException(AbstractUserDetailsAuthenticationProvider.this.messages  
             .getMessage("AbstractUserDetailsAuthenticationProvider.disabled", "User is disabled"));  
       }  
       if (!user.isAccountNonExpired()) {  
          AbstractUserDetailsAuthenticationProvider.this.logger  
             .debug("Failed to authenticate since user account has expired");  
          throw new AccountExpiredException(AbstractUserDetailsAuthenticationProvider.this.messages  
             .getMessage("AbstractUserDetailsAuthenticationProvider.expired", "User account has expired"));  
       }  
    }  
  
}
```

##### DefaultPostAuthenticationChecks
```Java
private class DefaultPostAuthenticationChecks implements UserDetailsChecker {  
  
    @Override  
    public void check(UserDetails user) {  
       if (!user.isCredentialsNonExpired()) {  //凭证是否过期
          AbstractUserDetailsAuthenticationProvider.this.logger  
             .debug("Failed to authenticate since user account credentials have expired");  
          throw new CredentialsExpiredException(AbstractUserDetailsAuthenticationProvider.this.messages  
             .getMessage("AbstractUserDetailsAuthenticationProvider.credentialsExpired",  
                   "User credentials have expired"));  
       }  
    }  
  
}
```

##### additionalAuthenticationChecks

```
protected abstract void additionalAuthenticationChecks(UserDetails userDetails,  
       UsernamePasswordAuthenticationToken authentication) throws AuthenticationException;
```

做一些凭证相关的校验

该方法在[[Spring/Spring_Security_01#DaoAuthenticationProvider\|DaoAuthenticationProvider]]有该方法的唯一实现
```Java
@Override  
@SuppressWarnings("deprecation")  
protected void additionalAuthenticationChecks(UserDetails userDetails,  
       UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {  
    if (authentication.getCredentials() == null) {  //判断是否存在凭证
       this.logger.debug("Failed to authenticate since no credentials provided");  
       throw new BadCredentialsException(this.messages  
          .getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));  
    }  
    //得到用户前端输入的未加密的raw密码
    String presentedPassword = authentication.getCredentials().toString();  
    //进行用户密码校验
    if (!this.passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {  
       this.logger.debug("Failed to authenticate since password does not match stored value");  
       throw new BadCredentialsException(this.messages  
          .getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));  
    }  
}
```

其中的PasswordEncoder为密码加密类
```Java
public interface PasswordEncoder {  
    String encode(CharSequence rawPassword);  
  
    boolean matches(CharSequence rawPassword, String encodedPassword);  
  
    default boolean upgradeEncoding(String encodedPassword) {  
        return false;  
    }  
}
```

#### 5️⃣createSuccessAuthentication

```Java
protected Authentication createSuccessAuthentication(Object principal, Authentication authentication,UserDetails user) {  
	//得到一个UsernamePasswordAuthenticationToken
	UsernamePasswordAuthenticationToken result = UsernamePasswordAuthenticationToken.authenticated(principal,authentication.getCredentials(),
	this.authoritiesMapper.mapAuthorities(user.getAuthorities()));  
	result.setDetails(authentication.getDetails());  
    this.logger.debug("Authenticated user");  
    return result;  
}
```

##### authoritiesMapper
this.authoritiesMapper默认为空实现(authorities原样返回)
```
private GrantedAuthoritiesMapper authoritiesMapper = new NullAuthentiesMapper()；
```
也可以自己定制GrantedAuthoritiesMapper

##### [[Spring/Spring_Security_01#DaoAuthenticationProvider\|DaoAuthenticationProvider]]中的createSuccessAuthentication
```Java
protected Authentication createSuccessAuthentication(Object principal, Authentication authentication, UserDetails user) {  
	//是否升级加密
    boolean upgradeEncoding = this.userDetailsPasswordService != null  
          && this.passwordEncoder.upgradeEncoding(user.getPassword());  
    if (upgradeEncoding) {  
       String presentedPassword = authentication.getCredentials().toString();  
       String newPassword = this.passwordEncoder.encode(presentedPassword);  
       user = this.userDetailsPasswordService.updatePassword(user, newPassword);  
    }  
    return super.createSuccessAuthentication(principal, authentication, user);  
}
```
##### userDetailsPasswordService
可以通过实现该接口提供updatePassword方法来自定义更新密码
#### 6️⃣retrieveUser🌟

##### 在[[Spring/Spring_Security_01#DaoAuthenticationProvider\|DaoAuthenticationProvider]]中唯一实现
```Java
@Override  
protected final UserDetails retrieveUser(String username, UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {  
    prepareTimingAttackProtection();  
    try {  
	    //通过loadUserByUsername拿到UserDetails
       UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);  
       if (loadedUser == null) {  
          throw new InternalAuthenticationServiceException(  
                "UserDetailsService returned null, which is an interface contract violation");  
       }  
       return loadedUser;  
    }  
    //可以自定义getUserDetailsService()，在loadedUser == null的时候抛出UsernameNotFoundException
    catch (UsernameNotFoundException ex) {  
       mitigateAgainstTimingAttack(authentication);  
       throw ex;  
    }  
    catch (InternalAuthenticationServiceException ex) {  
       throw ex;  
    }  
    catch (Exception ex) {  
       throw new InternalAuthenticationServiceException(ex.getMessage(), ex);  
    }  
}
```

##### prepareTimingAttackProtection()和mitigateAgainstTimingAttack(authentication)
用于干扰计时攻击，防止通过反应时间长短来判断用户名是否存在
计时攻击：密码匹配需要一个明显的耗时，攻击者可以通过耗时长短来得到一些测试信息。
#### 流程
先从缓存中获取，如果获取不到就调用retrieveUser方法获取，然后进行一系列检验，最后包装为createSuccessAuthentication对象返回

## UsernamePasswordAuthenticationFilter

### 父类为AbstractAuthenticationProcessingFilter
![Pasted image 20231107172100.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231107172100.png)
#### 父类的doFilter方法
```Java
private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {  
	
    if (!requiresAuthentication(request, response)) {  //是否需要认证
       chain.doFilter(request, response);  
       return;  
    }  
    try {  
	    //拿到的是已经认证成功的authentication
       Authentication authenticationResult = 2️⃣attemptAuthentication(request, response);  
       if (authenticationResult == null) {  
          // return immediately as subclass has indicated that it hasn't completed  
          return;  
       }  
       this.sessionStrategy.onAuthentication(authenticationResult, request, response);  
       // Authentication success  
       if (this.continueChainBeforeSuccessfulAuthentication) {  
          chain.doFilter(request, response);  //继续调用过滤器
       }  
	    //1️⃣successfulAuthentication中会调用successHandler，即自定义认证成功的处理逻辑
       successfulAuthentication(request, response, chain, authenticationResult);  

    }  
    catch (InternalAuthenticationServiceException failed) {  
       this.logger.error("An internal error occurred while trying to authenticate the user.", failed);  
       //登录失败调用failureHandler
       unsuccessfulAuthentication(request, response, failed);  
    }  
    catch (AuthenticationException ex) {  
       // Authentication failed  
       unsuccessfulAuthentication(request, response, ex);  
    }  
}
```

##### 1️⃣successfulAuthentication
```Java
protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authResult) throws IOException, ServletException {  
    SecurityContext context = this.securityContextHolderStrategy.createEmptyContext();  
    context.setAuthentication(authResult);  
    this.securityContextHolderStrategy.setContext(context);  
    this.securityContextRepository.saveContext(context, request, response);  
    if (this.logger.isDebugEnabled()) {  
       this.logger.debug(LogMessage.format("Set SecurityContextHolder to %s", authResult));  
    }  
    this.rememberMeServices.loginSuccess(request, response, authResult);  
    //发布事件
    if (this.eventPublisher != null) {  
       this.eventPublisher.publishEvent(new InteractiveAuthenticationSuccessEvent(authResult, this.getClass()));  
    }  
    //调用successHandler，即自定义认证成功的处理逻辑
    this.successHandler.onAuthenticationSuccess(request, response, authResult);  
}
```
### 2️⃣attemptAuthentication

>request进来 -> 包装为UsernamePasswordAuthenticationToken -> 传给[[Spring/Spring_Security_01#AuthenticationManager\|AuthenticationManager]]进行认证

```Java
@Override  
public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)  
       throws AuthenticationException {  
    if (this.postOnly && !request.getMethod().equals("POST")) {  
       throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());  
    }  
    String username = obtainUsername(request);  
    username = (username != null) ? username.trim() : "";  
    String password = obtainPassword(request);  
    password = (password != null) ? password : "";
    //拿到username和password之后，封装UsernamePasswordAuthenticationToken
    UsernamePasswordAuthenticationToken authRequest = UsernamePasswordAuthenticationToken.unauthenticated(username,  
          password);  
    // Allow subclasses to set the "details" property 
    //details：存储一些额外的信息 
    setDetails(request, authRequest);  
    return this.getAuthenticationManager().authenticate(authRequest);  
}
```

### 更改usernameParameter和passwordParameter

在UsernamePasswordAuthenticationFilter中规定了
```Java
public static final String SPRING_SECURITY_FORM_USERNAME_KEY = "username";  
  
public static final String SPRING_SECURITY_FORM_PASSWORD_KEY = "password";  

private String usernameParameter = SPRING_SECURITY_FORM_USERNAME_KEY;  
  
private String passwordParameter = SPRING_SECURITY_FORM_PASSWORD_KEY;
```

更改方法：
可以在我们写的config类中自定义：

```Java
public DefaultSecurityFilterChain securityFilterChain(HttpSecurity http) {
	http.formLogin()
	.usernameParameter("")
	.passwordParameter("")
	......
}
```

原因：formLogin()提供了一个[[Spring/Spring_Security_01#FormLoginConfigurer\|FormLoginConfigurer]]

```Java
public FormLoginConfigurer<HttpSecurity> formlogin() throws Exception {
	return getOrApply(new FormLoginConfigurer<>());
}
```

## FormLoginConfigurer

![Pasted image 20231107213158.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231107213158.png)

```Java
public final class FormLoginConfigurer<H extends HttpSecurityBuilder<H>> extends AbstractAuthenticationFilterConfigurer<H, FormLoginConfigurer<H>, UsernamePasswordAuthenticationFilter> {  
	public FormLoginConfigurer() {  
       super(new UsernamePasswordAuthenticationFilter(), null);  
       usernameParameter("username");  
       passwordParameter("password");  
    } 
     
    //登录页面
     @Override  
    public FormLoginConfigurer<H> loginPage(String loginPage) {  
       return super.loginPage(loginPage);  
    }  
    public FormLoginConfigurer<H> usernameParameter(String usernameParameter) {  
       getAuthenticationFilter().setUsernameParameter(usernameParameter);  
       return this;  
    }  
  
    public FormLoginConfigurer<H> passwordParameter(String passwordParameter) {  
       getAuthenticationFilter().setPasswordParameter(passwordParameter);  
       return this;  
    }  

	//失败重定向
    public FormLoginConfigurer<H> failureForwardUrl(String forwardUrl) {  
       failureHandler(new ForwardAuthenticationFailureHandler(forwardUrl));  
       return this;  
    }  

    public FormLoginConfigurer<H> successForwardUrl(String forwardUrl) {  
       successHandler(new ForwardAuthenticationSuccessHandler(forwardUrl));  
       return this;  
    }  
  
    @Override  
    public void init(H http) throws Exception {  
       super.init(http);  
       initDefaultLoginFilter(http);  
    }  
  
    @Override  
    protected RequestMatcher createLoginProcessingUrlMatcher(String loginProcessingUrl) {  
       return new AntPathRequestMatcher(loginProcessingUrl, "POST");  
    }  
  
    private String getUsernameParameter() {  
       return getAuthenticationFilter().getUsernameParameter();  
    }  
  
    private String getPasswordParameter() {  
       return getAuthenticationFilter().getPasswordParameter();  
    }  

	//在init方法中被调用
    private void initDefaultLoginFilter(H http) {  
       DefaultLoginPageGeneratingFilter loginPageGeneratingFilter = http  
          .getSharedObject(DefaultLoginPageGeneratingFilter.class);  
       if (loginPageGeneratingFilter != null && !isCustomLoginPage()) {  
          loginPageGeneratingFilter.setFormLoginEnabled(true);  
          loginPageGeneratingFilter.setUsernameParameter(getUsernameParameter());  
          loginPageGeneratingFilter.setPasswordParameter(getPasswordParameter());  
          loginPageGeneratingFilter.setLoginPageUrl(getLoginPage());  
          loginPageGeneratingFilter.setFailureUrl(getFailureUrl());  
          loginPageGeneratingFilter.setAuthenticationUrl(getLoginProcessingUrl());  
       }  
    }  
}
```

### 父类SecurityConfigurerAdapter

```Java
public abstract class SecurityConfigurerAdapter<O, B extends SecurityBuilder<O>> implements SecurityConfigurer<O, B> {  
  
    private B securityBuilder;  

	//属于configurer的objectPostProcessor，和securityBuilder中的objectPostProcessor不是同一个对象，但是作用相似
    private CompositeObjectPostProcessor objectPostProcessor = new CompositeObjectPostProcessor();  
  
    @Override  
    public void init(B builder) throws Exception {  
    }  
  
    @Override  
    public void configure(B builder) throws Exception {  
    }  
    //返回builder，可以达到链式调用的效果
    public B and() {  
       return getBuilder();  
    }  
  
	protected final B getBuilder() {  
       Assert.state(this.securityBuilder != null, "securityBuilder cannot be null");  
       return this.securityBuilder;  
    }  
 
    protected <T> T postProcess(T object) {  
       return (T) this.objectPostProcessor.postProcess(object);  
    }  
  
    public void addObjectPostProcessor(ObjectPostProcessor<?> objectPostProcessor) {  
       this.objectPostProcessor.addObjectPostProcessor(objectPostProcessor);  
    }  
  
	public void setBuilder(B builder) {  
       this.securityBuilder = builder;  
    }  
  
	private static final class CompositeObjectPostProcessor implements ObjectPostProcessor<Object> {  
		//可以维护多个postProcessor
       private List<ObjectPostProcessor<?>> postProcessors = new ArrayList<>();  
  
       @Override  
       @SuppressWarnings({ "rawtypes", "unchecked" })  
       //逐个调用
       public Object postProcess(Object object) {  
          for (ObjectPostProcessor opp : this.postProcessors) {  
             Class<?> oppClass = opp.getClass();  
             Class<?> oppType = GenericTypeResolver.resolveTypeArgument(oppClass, ObjectPostProcessor.class);  
             if (oppType == null || oppType.isAssignableFrom(object.getClass())) {  
                object = opp.postProcess(object);  
             }  
          }  
          return object;  
       }  
  
		private boolean addObjectPostProcessor(ObjectPostProcessor<?> objectPostProcessor) {  
		    boolean result = this.postProcessors.add(objectPostProcessor);  
	        this.postProcessors.sort(AnnotationAwareOrderComparator.INSTANCE);  
	        return result;  
	    }  
  
    }  
  
}
```

### 父类AbstractHttpConfigurer

#### disable

移除Configurer
```
public B disable() {  
    getBuilder().removeConfigurer(getClass());  
    return getBuilder();  
}
```

removeConfigurer方法在接口HttpSecurityBuilder中声明，在[[Spring/Spring_Security_01#AbstractConfiguredSecurityBuilder\|AbstractConfiguredSecurityBuilder]]中实现
```Java
public <C extends SecurityConfigurer<O, B>> C removeConfigurer(Class<C> clazz) {  
    List<SecurityConfigurer<O, B>> configs = this.configurers.remove(clazz);  
    if (configs == null) {  
       return null;  
    }  
    removeFromConfigurersAddedInInitializing(clazz);  
    Assert.state(configs.size() == 1,  
          () -> "Only one configurer expected for type " + clazz + ", but got " + configs);  
    return (C) configs.get(0);  
}
```

#### withObjectPostProcessor

添加一个objectPostProcessor
```Java
public T withObjectPostProcessor(ObjectPostProcessor<?> objectPostProcessor) {  
    addObjectPostProcessor(objectPostProcessor);  
    return (T) this;  
}
```

### 父类AbstractAuthenticationFilterConfigurer

实现了[[Spring/Spring_Security_01#SecurityConfigurer\|SecurityConfigurer]]，因此实现了init方法和configure方法

#### init

```Java
@Override  
public void init(B http) throws Exception {  
	//注册一些默认属性
    updateAuthenticationDefaults();  
    updateAccessDefaults(http);  
    registerDefaultAuthenticationEntryPoint(http);  
}
```

#### Configure

```Java
@Override  
public void configure(B http) throws Exception {  
    PortMapper portMapper = http.getSharedObject(PortMapper.class);  
    if (portMapper != null) {  
       this.authenticationEntryPoint.setPortMapper(portMapper);  
    }  
    //拿出共享对象，得到请求的缓存器
    RequestCache requestCache = http.getSharedObject(RequestCache.class);  
    if (requestCache != null) {  //1️⃣
       this.defaultSuccessHandler.setRequestCache(requestCache);  
    }  
    this.authFilter.setAuthenticationManager(http.getSharedObject(AuthenticationManager.class));  
    this.authFilter.setAuthenticationSuccessHandler(this.successHandler);  
    this.authFilter.setAuthenticationFailureHandler(this.failureHandler);  
    if (this.authenticationDetailsSource != null) {  
       this.authFilter.setAuthenticationDetailsSource(this.authenticationDetailsSource);  
    }  
    SessionAuthenticationStrategy sessionAuthenticationStrategy = http  
       .getSharedObject(SessionAuthenticationStrategy.class);  
    if (sessionAuthenticationStrategy != null) {  
       this.authFilter.setSessionAuthenticationStrategy(sessionAuthenticationStrategy);  
    }  
    RememberMeServices rememberMeServices = http.getSharedObject(RememberMeServices.class);  
    if (rememberMeServices != null) {  
       this.authFilter.setRememberMeServices(rememberMeServices);  
    }  
    //设置安全上下文
    SecurityContextConfigurer securityContextConfigurer = http.getConfigurer(SecurityContextConfigurer.class);  
    if (securityContextConfigurer != null && securityContextConfigurer.isRequireExplicitSave()) {  
       SecurityContextRepository securityContextRepository = securityContextConfigurer  
          .getSecurityContextRepository();  
       this.authFilter.setSecurityContextRepository(securityContextRepository);  
    }  
    this.authFilter.setSecurityContextHolderStrategy(getSecurityContextHolderStrategy());  
    F filter = postProcess(this.authFilter);  //2️⃣
    //添加filter
    http.addFilter(filter);  //3️⃣
}
```

##### 1️⃣this.defaultSuccessHandler.setRequestCache(requestCache)
保存用户登录失败前的页面，下次登录成功后默认打开上次登录的最后页面（即保存的页面），用户体验好

##### 2️⃣postProcess(this.authFilter)

###### 可以在我们配置的config类自定义postProcesser
```Java
public DefaultSecurityFilterChain securityFilterChain(HttpSecurity http) {
	http.formLogin()
	.withObjectPostProcessor(...)
	.addObjectPostProcessor(...)
	......
}
```
在...处new一个自己的objectPostProcessor，instanceof某一个filter

###### this.authFilter注入的时机

```
protected final void setAuthenticationFilter(F authFilter) {  
    this.authFilter = authFilter;  
}
```
其中setAuthenticationFilter被调用的时机为：
>在子类中被调用。
>例如子类FormLoginConfigurer
```Java
public FormLoginConfigurer() {  
    super(new UsernamePasswordAuthenticationFilter(), null);  
    usernameParameter("username");  
    passwordParameter("password");  
}
```
>其中super的构造方法
```Java
protected AbstractAuthenticationFilterConfigurer(F authenticationFilter, String defaultLoginProcessingUrl) {  
    this();  
    this.authFilter = authenticationFilter;  
    if (defaultLoginProcessingUrl != null) {  
       loginProcessingUrl(defaultLoginProcessingUrl);  
    }  
}
```

##### 3️⃣addFilter

HttpSecurityBuilder接口中声明，[[Spring/Spring_Security_01#HttpSecurity\|HttpSecurity]]中实现
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

## 匿名登录

>登录成功后，默认的用户名和密码由**SecurityProperties**类中的User类中的属性确定
```
private String name = "user";  
private String password = UUID.randomUUID().toString();
```


>默认情况下会执行UserDetailsServiceAutoConfiguration类中的：

- inMemoryUserDetailsManager方法
```Java
@Bean  
public InMemoryUserDetailsManager inMemoryUserDetailsManager(SecurityProperties properties, ObjectProvider<PasswordEncoder> passwordEncoder) {  
    SecurityProperties.User user = properties.getUser();  
    List<String> roles = user.getRoles();  
    return new InMemoryUserDetailsManager(new UserDetails[]{User.withUsername(user.getName()).password(this.getOrDeducePassword(user, (PasswordEncoder)passwordEncoder.getIfAvailable())).roles(StringUtils.toStringArray(roles)).build()});  
}
```
InMemoryUserDetailsManager类
![Pasted image 20231108201316.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231108201316.png)

- getOrDeducePassword方法
```Java
private String getOrDeducePassword(SecurityProperties.User user, PasswordEncoder encoder) {  
    String password = user.getPassword();  
    if (user.isPasswordGenerated()) {  
        logger.warn(String.format("%n%nUsing generated security password: %s%n%nThis generated password is for development use only. Your security configuration must be updated before running your application in production.%n", user.getPassword()));  
    }  
  
    return encoder == null && !PASSWORD_ALGORITHM_PATTERN.matcher(password).matches() ? "{noop}" + password : password;  
}
```


### 登录过程

进入[[Spring/Spring_Security_01#父类为AbstractAuthenticationProcessingFilter\|AbstractAuthenticationProcessingFilter]]的[[Spring/Spring_Security_01#父类的doFilter方法\|doFilter]]方法
	前端登录成功后会调用[[Spring/Spring_Security_01#UsernamePasswordAuthenticationFilter\|UsernamePasswordAuthenticationFilter]]的[[Spring/Spring_Security_01#2️⃣attemptAuthentication\|attemptAuthentication]]方法
	其中的this.getAuthenticationManager()是一个providerManager，但是该providerManage的providers属性中只有一个AnonymousAuthenticationProvider；而该providerManage的parent属性中的providers中有[[Spring/Spring_Security_01#DaoAuthenticationProvider\|DaoAuthenticationProvider]]
		然后执行[[Spring/Spring_Security_01#ProviderManager\|ProviderManager]]的authenticate方法，在判断子类provider是否支持的时候发现子类的唯一的AnonymousAuthenticationProvider不支持，然后调用父类的provider，发现支持[[Spring/Spring_Security_01#DaoAuthenticationProvider\|DaoAuthenticationProvider]]
			然后执行provider.[[Spring/Spring_Security_01#🌟authenticate方法\|authenticate]]方法，进入[[Spring/Spring_Security_01#AbstractUserDetailsAuthenticationProvider\|AbstractUserDetailsAuthenticationProvider]]类
				然后进入执行[[Spring/Spring_Security_01#6️⃣retrieveUser🌟\|retrieveUser]]方法，进入[[Spring/Spring_Security_01#DaoAuthenticationProvider\|DaoAuthenticationProvider]]类，
				其中的this.getUserDetailsService为inMemoryUserDetailsManager
			执行到[[Spring/Spring_Security_01#AbstractUserDetailsAuthenticationProvider\|AbstractUserDetailsAuthenticationProvider]]的[[Spring/Spring_Security_01#🌟authenticate方法\|authenticate]]方法的return createSuccessAuthentication语句时
				执行方法createSuccessAuthentication，进入[[Spring/Spring_Security_01#DaoAuthenticationProvider\|DaoAuthenticationProvider]]类
				执行到return super.createSuccessAuthentication的时候
					进入父类[[Spring/Spring_Security_01#AbstractUserDetailsAuthenticationProvider\|AbstractUserDetailsAuthenticationProvider]]的[[Spring/Spring_Security_01#5️⃣createSuccessAuthentication\|createSuccessAuthentication]]方法
					执行UsernamePasswordAuthenticationToken.authenticated方法得到UsernamePasswordAuthenticationToken
			*......一系列返回*
执行完attemptAuthentication方法，返回认证成功的[[Spring/Spring_Security_01#Authentication\|Authentication]]到[[Spring/Spring_Security_01#父类为AbstractAuthenticationProcessingFilter\|AbstractAuthenticationProcessingFilter]]，继续完成[[Spring/Spring_Security_01#父类的doFilter方法\|doFilter]]方法
	调用successfulAuthentication方法
		调用onAuthenticationSuccess方法，进入SavedRequestAwareAuthenticationSuccessHandler类，进行重定向
