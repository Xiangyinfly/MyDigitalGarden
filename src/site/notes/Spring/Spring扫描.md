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
componentScan.getBoolean("useDefaultFilters")：默认为true，即使用默认的过滤器

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

#### 6.扫描包路径

```
Set<String> basePackages = new LinkedHashSet<>();
		String[] basePackagesArray = componentScan.getStringArray("basePackages");//可以传入字符串数组
		for (String pkg : basePackagesArray) {
			String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
					ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);//按照规则解析
			Collections.addAll(basePackages, tokenized);
		}
		for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
			basePackages.add(ClassUtils.getPackageName(clazz));
		}
		//空处理
		if (basePackages.isEmpty()) {
			basePackages.add(ClassUtils.getPackageName(declaringClass));
		}
```

basePackageClasses属性：将传入类的包名作为路径传入
空处理：如果未传入任何参数，就将添加该注解的config类的包路径作为扫描路径

#### 7.排除config类

```
scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
			@Override
			protected boolean matchClassName(String className) {
				return declaringClass.equals(className);
			}
		});
```

ExcludeFilter的作用：
排除添加改注解的config类。因为config类在AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class)语句中已经手动加入spring容器
### 3️⃣：return scanner.doScan(包路径)

#### doScan方法
```
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {  
    Assert.notEmpty(basePackages, "At least one base package must be specified");  
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();  
    for (String basePackage : basePackages) { 
		//真正进行扫描，扫描得到的结果为BeanDefinition
       Set<BeanDefinition> candidates = 1️⃣findCandidateComponents(basePackage);  
       for (BeanDefinition candidate : candidates) {  
          ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);  
          candidate.setScope(scopeMetadata.getScopeName());  
          String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);  
          if (candidate instanceof AbstractBeanDefinition abstractBeanDefinition) {  
             postProcessBeanDefinition(abstractBeanDefinition, beanName);  
          }  
          if (candidate instanceof AnnotatedBeanDefinition annotatedBeanDefinition) {  
             AnnotationConfigUtils.processCommonDefinitionAnnotations(annotatedBeanDefinition);  
          }  

		//检查是否扫描到相同的BeanDefinition。不同包可能扫描到相同的bean
          if (5️⃣checkCandidate(beanName, candidate)) {  
             BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);  
             definitionHolder =  
                   AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);  
             beanDefinitions.add(definitionHolder);  
             //在spring容器中注册
             registerBeanDefinition(definitionHolder, this.registry);  
          }  
       }  
    }  
    return beanDefinitions;  
}
```

##### 5️⃣检测是否存在相同名称的bean：checkCandidate(beanName, candidate)
```
protected boolean checkCandidate(String beanName, BeanDefinition beanDefinition) throws IllegalStateException {  
    if (!this.registry.containsBeanDefinition(beanName)) {  
       return true;  
    }  
    BeanDefinition existingDef = this.registry.getBeanDefinition(beanName);  
    BeanDefinition originatingDef = existingDef.getOriginatingBeanDefinition();  
    if (originatingDef != null) {  
       existingDef = originatingDef;  
    }  
    if (6️⃣isCompatible(beanDefinition, existingDef)) {  //判断是否兼容
       return false;  
    }  
    //抛异常，spring容器不能正常启动
    throw new ConflictingBeanDefinitionException("Annotation-specified bean name '" + beanName +  
          "' for bean class [" + beanDefinition.getBeanClassName() + "] conflicts with existing, " +  
          "non-compatible bean definition of same name and class [" + existingDef.getBeanClassName() + "]");  
}
```

###### 6️⃣判断是否兼容：isCompatible(beanDefinition, existingDef))
```
protected boolean isCompatible(BeanDefinition newDef, BeanDefinition existingDef) { 
	//两个类不一样，返回false->导致抛异常
    return (!(existingDef instanceof ScannedGenericBeanDefinition) ||  // explicitly registered overriding bean  
          (newDef.getSource() != null && newDef.getSource().equals(existingDef.getSource())) ||  // scanned same file twice  
          newDef.equals(existingDef));  // scanned equivalent class twice  
}
```

##### 1️⃣真正进行扫描的方法：findCandidateComponents
```
public Set<BeanDefinition> findCandidateComponents(String basePackage) {  
	//7️⃣优化的方法
    if (this.componentsIndex != null && indexSupportsIncludeFilters()) {  
       return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);  
    }  
    else {  
	    //一般执行else
       return 2️⃣scanCandidateComponents(basePackage);  
    }  
}
```

###### 7️⃣componentsIndex机制

在spring.components文件中指定要成为bean的类。
spring只会去扫描spring.components文件中指定的类并生成bean。

```
com.example.service.Service1=org.springframework.stereotype.Component
```

###### 2️⃣else的方法：scanCandidateComponents
```
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {  
    Set<BeanDefinition> candidates = new LinkedHashSet<>();  
    try {  
       String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +  
             resolveBasePackage(basePackage) + '/' + this.resourcePattern;  //获取完整的路径
       Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);//获取扫描路径下面的所有.class文件资源  
       boolean traceEnabled = logger.isTraceEnabled();  
       boolean debugEnabled = logger.isDebugEnabled();  
       for (Resource resource : resources) {  
          String filename = resource.getFilename();  
          if (filename != null && filename.contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {  
             // Ignore CGLIB-generated classes in the classpath  
             continue;  
          }  
          if (traceEnabled) {  
             logger.trace("Scanning " + resource);  
          }  
          try {  
             MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource); //得到类的元数据读取器 
             if (3️⃣isCandidateComponent(metadataReader)) {  //进行第一次判断
                ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);  //判断完成后生成beanDefinition
                sbd.setSource(resource);  
                if (4️⃣isCandidateComponent(sbd)) {  //进行第二次判断
                   if (debugEnabled) {  
                      logger.debug("Identified candidate component class: " + resource);  
                   }  
                   candidates.add(sbd);  //满足条件后加入集合返回
                }  
                else {  
                   if (debugEnabled) {  
                      logger.debug("Ignored because not a concrete top-level class: " + resource);  
                   }  
                }  
             }  
             else {  
                if (traceEnabled) {  
                   logger.trace("Ignored because not matching any filter: " + resource);  
                }  
             }  
          }  
          catch (FileNotFoundException ex) {  
             if (traceEnabled) {  
                logger.trace("Ignored non-readable " + resource + ": " + ex.getMessage());  
             }  
          }  
          catch (Throwable ex) {  
             throw new BeanDefinitionStoreException(  
                   "Failed to read candidate component class: " + resource, ex);  
          }  
       }  
    }  
    catch (IOException ex) {  
       throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);  
    }  
    return candidates;  
}
```

>问题：如何得到扫描路径下的类的metadata？

>解决：
>如果使用反射，会在spring容器加载的时候将扫描路径下的所有类加载到jvm中，如果所有这些类中只有极少一部分最终需要成为bean，会造成很大的资源浪费
>使用asm技术，不涉及到类的加载就可以得到类的信息


3️⃣第一次判断isCandidateComponent(metadataReader)：
```
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {  
    for (TypeFilter tf : this.excludeFilters) {  
	    //如果该类信息与其中一个excludeFilters匹配，则返回false，该类会被排除掉
       if (tf.match(metadataReader, getMetadataReaderFactory())) {  
          return false;  
       }  
    }  
    for (TypeFilter tf : this.includeFilters) {  
	    //至少匹配一个includeFilters，才有可能成为一个bean
       if (tf.match(metadataReader, getMetadataReaderFactory())) {  
          return isConditionMatch(metadataReader);  //判断有没有@Conditional注解且注解自定义的条件

       }  
    }  
    return false;  
}
```

>tf.match(metadataReader, getMetadataReaderFactory()):
>如果没有指定过滤器，则会使用默认过滤器。
>默认过滤器在new scanner的时候设置：componentScan.getBoolean("useDefaultFilters")：默认为true，即使用默认的过滤器
>在scanner的构造方法中，会调用方法设置三个includeFilter（三个注解对应的过滤器，@Component，@Named等）


>isConditionMatch(metadataReader)：
>判断有没有满足利用@Conditional注解自定义的条件。
>写一个配置类c实现Condition接口，重写matches方法， 在其中自定义bean创建的条件。service类中添加@Conditional(c.class)以生效。


4️⃣第二次判断isCandidateComponent(sbd)：
```
protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {  
    AnnotationMetadata metadata = beanDefinition.getMetadata();  
    return (metadata.isIndependent() && (metadata.isConcrete() ||  
          (metadata.isAbstract() && metadata.hasAnnotatedMethods(Lookup.class.getName()))));  
}
```

>metadata.isIndependent()：是否为独立类。非静态的内部类不是独立类，非静态的内部类实例化要依赖外部类的对象。非静态的内部类被排除不能生成bean
>静态的内部类实例化不依赖外部类的对象，可以生成bean

>metadata.isConcrete()：既不是接口也不是抽象类，返回true。接口和抽象类不能实例化，不能生成bean对象。mybatis中的mapper接口通过动态代理生成代理类的bean对象

>metadata.isAbstract() && metadata.hasAnnotatedMethods(Lookup.class.getName()：是抽象类且类中存在添加了Lookup注解的方法 ，也返回true
>sd
>独立的 且 类中存在添加了Lookup注解的方法 的 抽象类 生成的bean对象是其代理对象


## 关于@Lookup注解

>@Lookup注解的属性需要指定一个bean名字，@Lookup(bean名)

>添加了@Lookup注解的方法不会执行方法里的逻辑，只会返回bean名字对应的bean。
>因此返回值类型为bean对象的类型。

>问题：假设s1类是singleton，s2类是prototype且s1类中有属性s2。
>1.在s1中用普通方法直接打印多次s2得到的s2对象是同一个吗？
>2.用s1的添加了@Lookup注解的方法获取s2，得到的s2对象是同一个吗？

>答案：
>1.是同一个。因为s1中的s2属性在创建s1的bean的时候已经注入。
>2.不是同一个。