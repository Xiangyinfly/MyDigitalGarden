---
{"dg-publish":true,"permalink":"/spring/my-spring-mybatis/","dgPassFrontmatter":true}
---

## Mybatis代理对象

```
@Component  
public class UserService {  
    @Autowired  
    private UserMapper userMapper;  
  
    public void test() {  
        System.out.println(userMapper);  
    }  
}
```

>问题：想要拿到mybatis创建的代理对象并且赋值给userMapper属性，需要让Mybatis生成的mapper代理对象变成一个bean。如何做到？

>解决：
>写一个MyFactoryBean类实现FactoryBean类

```
@Component
public class MyFactoryBean implements FactoryBean {  
    @Override  
    public Object getObject() throws Exception {  
        Object instance = Proxy.newProxyInstance(MyFactoryBean.class.getClassLoader(), new Class[]{UserMapper.class}, new InvocationHandler() {  
            @Override  
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
                return null;  
            }  
        });  
  
        return instance;  
    }  
  
    @Override  
    public Class<?> getObjectType() {  
        return UserMapper.class;  
    }  
}
```

>调用applicationContext.getBean("myFactoryBean")返回的是getObject()方法返回的代理对象
>调用applicationContext.getBean("&myFactoryBean")返回的是MyFactoryBean本身的bean
>因此我们自己实现了创建mapper的代理对象，将其变成一个bean并赋值给service中的属性。

>那如何拿到mybatis创建的代理对象？

>解决：
>在MyFactoryBean里，
```
private SqlSession sqlSession;  
  
@Autowired  
public void setSqlSession(SqlSessionFactory sqlSessionFactory) {  
    sqlSessionFactory.getConfiguration().addMapper(UserMapper.class);  
    this.sqlSession = sqlSessionFactory.openSession();  
}  
  
@Override  
public Object getObject() throws Exception {  
    UserMapper mapper = sqlSession.getMapper(UserMapper.class);  
    return mapper;  
}
```

>通过SqlSessionFactory拿到sqlSession，再通过sqlSession拿到mb创建的代理对象。
>而SqlSessionFactory需要再config类中自己配置。
```
@Bean  
public SqlSessionFactory sqlSessionFactory() throws Exception {  
    InputStream is = Resources.getResourceAsStream("mybatis. xml");  
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(is);  
    return sqlSessionFactory;  
}
```


## 动态mapper


>问题：如何实现动态的mapper？

>解决：
>在MyFactoryBean添加属性private Class mapperClass，通过构造方法对mapperClass赋值

```
@Component  
public class MyFactoryBean implements FactoryBean {
	@Override  
	public Class<?> getObjectType() {  
	    return mapperClass;  
	}  
	  
	public MyFactoryBean(Class mapperClass) {  
	    this.mapperClass = mapperClass;  
	}  
	  
	private SqlSession sqlSession;  
	private Class mapperClass;  
	  
	@Autowired  
	public void setSqlSession(SqlSessionFactory sqlSessionFactory) {  
	    sqlSessionFactory.getConfiguration().addMapper(mapperClass);  
	    this.sqlSession = sqlSessionFactory.openSession();  
	}  
	  
	@Override  
	public Object getObject() throws Exception {  
	    //UserMapper mapper = sqlSession.getMapper(UserMapper.class);  
	    return sqlSession.getMapper(mapperClass);  
	}
}
```


>问题：但是方法返回的只有一个bean，我们想要所有的bean，如何实现？

>解决：
>使用beanDefinition

在main方法中：
```
AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();  
  
applicationContext.register(AppConfig.class);  

//得到beanDefinition
AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition().getBeanDefinition();  
beanDefinition.setBeanClass(MyFactoryBean.class);  
//通过该方法对构造器赋值，此处赋值为UserMapper.class，从而实现动态mapper
beanDefinition.getConstructorArgumentValues().addGenericArgumentValue(UserMapper.class);  
//注册beanDefinition ，定义一个叫UserMapper的bean
applicationContext.registerBeanDefinition("UserMapper",beanDefinition);  
applicationContext.refresh();
```

>问题：为什么要手动写register和refresh？

>答案：
>因为中间插入了自定义的beanDefinition，要将这两步单独拿出来执行

>问题：为什么beanDefinition.setBeanClass(MyFactoryBean.class)？

>答案：因为如果传入UserMapper.class，UserMapper是接口不会产生对象
>传入UserMapper.class后，会生成其对象。spring发现是一个FactoryBean，就会调用getObject方法，userMapper属性对应的代理对象作为另外一个bean对象被返回

*这几行会生成两个对象，一个MyFactoryBean的对象，一个UserMapper的代理对象
因为使用了beanDefinition，所以不用在MyFactoryBean类中加@Component注解类*

>问题：把得到beanDefinition的定义都写在main方法里很不美观，如何解决？

>解决：写一个类MyImportBeanDefinitionRegistrar实现ImportBeanDefinitionRegistrar接口，把定义代码写入重写的registerBeanDefinitions方法。

```
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {  
    @Override  
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry, BeanNameGenerator importBeanNameGenerator) {  
        AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition().getBeanDefinition();  
        beanDefinition.setBeanClass(MyFactoryBean.class);  
        beanDefinition.getConstructorArgumentValues().addGenericArgumentValue(UserMapper.class);  
  
        registry.registerBeanDefinition("UserMapper",beanDefinition);  
    }  
}
```

> 最后在config类中加入注解@Import(MyImportBeanDefinitionRegistrar.class)

## Mapper扫描

>问题：由于spring的FactoryBean和ImportBeanDefinitionRegistra类都封装在jar包里，我们无法在ImportBeanDefinitionRegistra类中写入自己的mapper，如何解决？

>解决：创建一个注解@MyMapperScan

```
@Retention(RetentionPolicy.RUNTIME)  
@Target(ElementType.TYPE)  
@Import(MyImportBeanDefinitionRegistrar.class)  
  
public @interface MyMapperScan {  
    String value() default "";  
}
```

>在config类引用注解@MyMapperScan("com.xiang.mapper")
>在MyImportBeanDefinitionRegistra的registerBeanDefinitions方法实现扫描所有mapper
```
@Override  
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry, BeanNameGenerator importBeanNameGenerator) {  
  
    Map<String, Object> annotationAttributes = importingClassMetadata.getAnnotationAttributes(MyMapperScan.class.getName());  
    String path = (String) annotationAttributes.get("value");  //根据注解得到mapper类的路径

	//接下来扫描mapper
	MyMapperScanner myMapperScanner = new MyMapperScanner(registry);  
	myMapperScanner.scan(path);

	//注册bean
    AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition().getBeanDefinition();  
    beanDefinition.setBeanClass(MyFactoryBean.class);  

	//给构造方法添加参数（重要！）
	beanDefinition.getConstructorArgumentValues()
	.addGenericArgumentValue(UserMapper.class); 
    registry.registerBeanDefinition("UserMapper",beanDefinition);
}
```

>新建一个扫描类MyMapperScanner继承spring创建bean的扫描类ClassPathBeanDefinitionScanner，用于扫描mapper

>问题：
>但是spring的扫描类不会扫描接口，因为接口无法创建bean。恰恰相反，mybatis的扫描需要扫描mapper接口，如何解决？

>在MyMapperScanner中重写isCandidateComponent，设置只关心接口；重写doScan，生成beanDefinition
```
public class MyMapperScanner extends ClassPathBeanDefinitionScanner {  
    public MyMapperScanner(BeanDefinitionRegistry registry) {  
        super(registry);  
    }  

	//设置只关心接口  
	@Override  
	protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {  
	    return beanDefinition.getMetadata().isInterface();  
	}  

	//生成beanDefinition
	@Override  
	protected Set<BeanDefinitionHolder> doScan(String... basePackages) {  
	    Set<BeanDefinitionHolder> beanDefinitionHolders = super.doScan(basePackages);  
	    return beanDefinitionHolders;
	}
}
```

>但是doScan方法里面的beanDefinitionHolders为空，可以在MyImportBeanDefinitionRegistrar的registerBeanDefinitions方法加入myMapperScanner.addIncludeFilter()解决（以后解释）
```
MyMapperScanner myMapperScanner = new MyMapperScanner(registry);  

myMapperScanner.addIncludeFilter(new TypeFilter() {  
    @Override  
    public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {  
        return true;  
    }  
});  
  
myMapperScanner.scan(path);
```

>问题：
>这样得到的beanDefinition类型是UserMapper.class，而我们设置的是MyFactoryBean.class

>解决：在MyMapperScanner中改写doScan方法，修改错误的beanDefinition类型

```
@Override  
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {  
    Set<BeanDefinitionHolder> beanDefinitionHolders = super.doScan(basePackages);  
  
    for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {  
        BeanDefinition beanDefinition = beanDefinitionHolder.getBeanDefinition();  
        //先设置构造方法参数  
        beanDefinition.getConstructorArgumentValues().addGenericArgumentValue(beanDefinition.getBeanClassName());  
        //修改错误的beanDefinition类型  
        beanDefinition.setBeanClassName(MyFactoryBean.class.getName());  
    }  
    return beanDefinitionHolders;  
}
```

此时可以直接通过在service中注解注入mapper来实现mapper代理对象的获取