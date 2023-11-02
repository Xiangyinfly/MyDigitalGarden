---
{"dg-publish":true,"permalink":"/spring/spring/","dgPassFrontmatter":true}
---


*扫描的结果是beanDefinition*

## @ComponentScan的扫描

解析ComponentScanAnnotationParser类：


### 1️⃣：new一个scanner
```
ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner (this.registry,componentScan.getBoolean("useDefaultFilters"), this. environment, this. resourceLoader);
```
this.registry：spring容器，用来放入产生的beanDefinition
componentScan.getBoolean("useDefaultFilters")：默认为true

### 2️⃣：扫描器scanner去设置一些属性

#### 1.自定义bean名字
```
	//自定义bean名字
	Class<? extends BeanNameGenerator> generatorClass = componentScan.getClass("nameGenerator");
	boolean useInheritedGenerator = (BeanNameGenerator.class == generatorClass);
	//判断是否为默认值
	scanner.setBeanNameGenerator(useInheritedGenerator ? this.beanNameGenerator :
				BeanUtils.instantiateClass(generatorClass));
```
默认的为AnnotationBeanNameGenerator，生成方法如下
```
@Override
public String generateBeanName (BeanDefinition definition,BeanDefinitionRegistry
registry) {
	if(definition instanceof AnnotatedBeanDefinition) {
	//获取注解所指定的beanName
	//默认为类@Component注解里value属性的值
		String beanName = determineBeanNameFromAnnotation((AnnotatedBeanDefinition) definition);
		if (StringUtils. hasText (beanName)) {
		// Explicit bean name found.
		return beanName;
	}
	// Fallback: generate a unique default bean name.
	//生成默认名
	return buildDefaultBeanName (definition, registry);
}
```
生成默认名的方法
```
protected String buildDefaultBeanName(BeanDefinition definition) {
	String beanClassName = definition. getBeanClassName;
	Assert. state ( beanClassName != null,"No bean class name set");
	String shortClassName = ClassUtils. getShortName (beanClassName) ;
	return Introspector. decapitalize(shortClassName);
｝
```


我们可以设置一个MyBeanNameGenerator类，实现BeanNameGenerator接口来自定义bean的名字
```
public class MyBeanNameGenerator implements BeanNameGenerator { 
	@Override
	public String generateBeanName(BeanDefinitiondefinition,BeanDefinitionRegistry
registry) {
		return "My"+definition.getBeanClassName();
	}
}
```
然后在config类的@Component注解添加属性nameGenerator = MyBeanNameGenerator.class)以起作用


#### 2.设置默认代理模式
```
ScopedProxyMode scopedProxyMode = componentScan.getEnum("scopedProxy");
		if (scopedProxyMode != ScopedProxyMode.DEFAULT) {
			scanner.setScopedProxyMode(scopedProxyMode);
		}
		else {
			Class<? extends ScopeMetadataResolver> resolverClass = componentScan.getClass("scopeResolver");
			scanner.setScopeMetadataResolver(BeanUtils.instantiateClass(resolverClass));
		}
```

关于scopedProxy属性：
前置知识： 假设s1类加上@Scope (WebApplicationContext. SCOPE_REQUEST)注解，即只有创建请求的时候才会生成bean。而s2类中有@Autowired的s1类（创建的时候强制要求注入s1），但是在s2创建的时候（即容器初始化的时候），s1并未被创建。spring解决该问题的方法是先生成s1的一个代理对象注入s2，而改代理对象的生成模式由@Scope注解中的proxyMode属性决定（cglib、jdk等）。
在@ComponentScan注解的scopeProxy属性中也可以批量配置类的代理对象生成模式。
如果类的@Scope注解中未配置代理模式，就使用@ComponentScan注解所配置的。

#### 3.资源匹配规则

```
scanner.setResourcePattern(componentScan.getString("resourcePattern"));
```

默认为static final String DEFAULT_RESOURCE_PATTERN = "\*\*/\*.class"；
即扫描传入路径及其子路径下的所有.class文件

#### 4.两个filter

```
for (AnnotationAttributes includeFilterAttributes : componentScan.getAnnotationArray("includeFilters")) {
			List<TypeFilter> typeFilters = TypeFilterUtils.createTypeFiltersFor(includeFilterAttributes, this.environment,
					this.resourceLoader, this.registry);
			for (TypeFilter typeFilter : typeFilters) {
				scanner.addIncludeFilter(typeFilter);
			}
		}
		for (AnnotationAttributes excludeFilterAttributes : componentScan.getAnnotationArray("excludeFilters")) {
			List<TypeFilter> typeFilters = TypeFilterUtils.createTypeFiltersFor(excludeFilterAttributes, this.environment,
				this.resourceLoader, this.registry);
			for (TypeFilter typeFilter : typeFilters) {
				scanner.addExcludeFilter(typeFilter);
			}
		}
```

>excludeFilter：
>excludeFilters = \{@ComponentScan.Filter (type = FilterType.ASSIGNABLE_TYPE,classes = UserService. class)}
>即排除扫描路径下的UserService. class文件，不创建UserService的bean

#### 5.配置懒加载
```
boolean lazyInit = componentScan.getBoolean("lazyInit");
		if (lazyInit) {
			scanner.getBeanDefinitionDefaults().setLazyInit(true);
		}
```


### 3️⃣：return scanner.doScan(包路径)


