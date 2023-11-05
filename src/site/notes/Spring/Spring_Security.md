---
{"dg-publish":true,"permalink":"/spring/spring-security/","dgPassFrontmatter":true}
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

![Pasted image 20231105145428.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231105145428.png)

```
public interface SecurityFilterChain {  
	//如果返回true，就会对request应用list里的过滤器
    boolean matches(HttpServletRequest request);  
  
    List<Filter> getFilters();  
  
}
```

实现类
```
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
其中有些方法不是abstract，说明这些方法不重要，子类不需要重写（必须实现）
由代码可见通过**performBuild**方法构建了result对象
performBuild方法有三个实现类，其中包括熟悉的HttpSecurity和WebSecurity 

>在HttpSecurity中，该方法 return new DefaultSecurityFilterChain(this.requestMatcher, sortedFilters); 

>在WebSecurity中，该方法返回了一个 [[Spring/Spring_Security#filterChainProxy\|FilterChainProxy]]
```
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

### FilterChainProxy

![Pasted image 20231105185837.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231105185837.png)

其构造方法传入了[[Spring/Spring_Security#SecurityFilterChain\|SecurityFilterChain]]的集合
```
public FilterChainProxy(List<SecurityFilterChain> filterChains) {  
    this.filterChains = filterChains;  
}
```

其中doFilter方法 -> doFilterInternal方法 
```
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
>filterChainProxy调用doFilter方法，再调用doFilterInternal方法，通过getFilters(HttpServletRequest request)方法通过传入的request得到第一个匹配的SecurityFilterChain，通过getFilters()方法得到SecurityFilterChain中所有的过滤器，构建为一个VirtualFilterChain，最后调用每个过滤器的doFilter方法。


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

## SecurityConfigurer

>SecurityBuilder用来构建一个对象，SecurityConfigurer用来配置一个SecurityBuilder
```
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

*AbstractConfiguredSecurityBuilder被SecurityConfigurer所配置 *

AbstractConfiguredSecurityBuilder类：
```
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
```
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