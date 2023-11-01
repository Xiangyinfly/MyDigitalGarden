---
{"dg-publish":true,"permalink":"/spring/spring/","dgPassFrontmatter":true}
---


*扫描的结果是beanDefinition*

## @ComponentScan的扫描

解析ComponentScanAnnotationParser类：


>1️⃣：
```
ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner (this.registry,componentScan.getBoolean("useDefaultFilters"), this. environment, this. resourceLoader);
```
this.registry：spring容器，用来放入产生的beanDefinition
componentScan.getBoolean("useDefaultFilters")：默认为true

>2️⃣：扫描器scanner去设置一些属性
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


>3️⃣：return scanner.doScan(包路径)


