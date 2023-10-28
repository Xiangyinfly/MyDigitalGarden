---
{"dg-publish":true,"permalink":"/spring/my-spring/","dgPassFrontmatter":true}
---

## 一些注解类

@ComponentScan
```
@Target({ElementType.TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
public @interface ComponentScan {  
    String value() default "";  
}
```

@Component
```
@Target({ElementType.TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
public @interface Component {  
    String value() default "";  
}
```

@Scope
```
@Target({ElementType.TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
public @interface Scope {  
    String value() default "";  
}
```

@Autowired
```
@Target({ElementType.FIELD})  
@Retention(RetentionPolicy.RUNTIME)  
public @interface Autowired {  
}
```

## 建立BeanDefinition

```
public class BeanDefinition {  
    private Class type;  
    private String scope;  
  
    public Class getType() {  
        return type;  
    }  
  
    public void setType(Class type) {  
        this.type = type;  
    }  
  
    public String getScope() {  
        return scope;  
    }  
  
    public void setScope(String scope) {  
        this.scope = scope;  
    }  
}
```

## MyApplicationContext类

```
public class MyApplicationContext {  
    private Class configClass;  
    private ConcurrentHashMap<String,BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>();  
    private ConcurrentHashMap<String,Object> singletonObjects = new ConcurrentHashMap<>();  
  
    public MyApplicationContext(Class configClass) {  
        this.configClass = configClass;  
  
        //扫描类的注解  
        if (configClass.isAnnotationPresent(ComponentScan.class)) {  
            ComponentScan componentScanAnnotation = (ComponentScan) configClass.getAnnotation(ComponentScan.class);  
            String path = componentScanAnnotation.value();  
            path = path.replace(".","/");  
  
            ClassLoader classLoader = MyApplicationContext.class.getClassLoader();  
            URL resource = classLoader.getResource(path);  
  
            File file = new File(resource.getFile());  
            System.out.println(file);  
            if (file.isDirectory()) {  
                File[] files = file.listFiles();  
                for (File f : files) {  
                    String fileName = f.getAbsolutePath();  
                    if (fileName.endsWith(".class")) {  
                        String className = (path + "/" + f.getName().substring(0,f.getName().indexOf(".class"))).replace("/",".");  
  
                        try {  
                            Class<?> clazz = classLoader.loadClass(className);  
  
  
                            if (clazz.isAnnotationPresent(Component.class)) {  
                                String beanName = clazz.getAnnotation(Component.class).value();  
                                if ("".equals(beanName)) {  
                                    beanName = Introspector.decapitalize(clazz.getSimpleName());  
                                }  
  
                                BeanDefinition beanDefinition = new BeanDefinition();  
                                beanDefinition.setType(clazz);  
                                if (clazz.isAnnotationPresent(Scope.class)) {  
                                    Scope scopeAnnotation = clazz.getAnnotation(Scope.class);  
                                    beanDefinition.setScope(scopeAnnotation.value());  
                                } else {  
                                    beanDefinition.setScope("singleton");  
                                }  
  
                                beanDefinitionMap.put(beanName,beanDefinition);  
                            }  
                        } catch (ClassNotFoundException e) {  
                            throw new RuntimeException(e);  
                        }  
                    }  
                }  
            }  
        }  
  
        //create所有的单例bean  
        beanDefinitionMap.keySet().forEach(beanName -> {  
            BeanDefinition beanDefinition = beanDefinitionMap.get(beanName);  
            if (beanDefinition.getScope().equals("singleton")) {  
                Object bean = createBean(beanName,beanDefinition);  
                singletonObjects.put(beanName,bean);  
            }  
        });  
    }  
	//创建bean
    public Object createBean(String beanName,BeanDefinition beanDefinition) {  
        Class clazz = beanDefinition.getType();  
        try {  
            Object instance = clazz.getConstructor().newInstance();  
  
            //简易DI  
            for (Field f : clazz.getDeclaredFields()) {  
                if (f.isAnnotationPresent(Autowired.class)) {  
                    f.setAccessible(true);  
                    //把带autowired注解的字段的实例设置为getBean(f.getName())  
                    f.set(instance,getBean(f.getName()));  
                }  
            }  
  
            
            return instance;  
        } catch (InstantiationException e) {  
            throw new RuntimeException(e);  
        } catch (IllegalAccessException e) {  
            throw new RuntimeException(e);  
        } catch (InvocationTargetException e) {  
            throw new RuntimeException(e);  
        } catch (NoSuchMethodException e) {  
            throw new RuntimeException(e);  
        }  
    }  
	//getBean方法实现
    public Object getBean(String beanName) {  
        BeanDefinition beanDefinition = beanDefinitionMap.get(beanName);  
        if (beanDefinition == null) {  
            throw new NullPointerException();  
        }  
  
        String scope = beanDefinition.getScope();  
  
        //单例  
        if (scope.equals("singleton")) {  
            Object bean = singletonObjects.get(beanName);  
  
            /*  
            此处空处理的原因：比如，在创建UserService的bean的时候，createBean方法中会进行依赖注入OrderService，  
            而此时会调用getBean方法（f.set(instance,getBean(f.getName()));），但这时可能单例库里还没有OrderService，  
            所以要进行空处理并且将其加入单例库  
             */ 
            //未解决循环依赖问题           
            if (bean == null) {  
                bean = createBean(beanName,beanDefinition);  
            }  
            return bean;  
        }  
  
        return createBean(beanName,beanDefinition);  
    }  
}
```
## Aware回调

在spring文件夹中创建接口BeanNameAware

```
public interface BeanNameAware {
    void setBeanName(String beanName);
}
```

MyApplicationContext中的createBean方法写入

```
public Object createBean(String beanName,BeanDefinition beanDefinition) {
    ...
        //beanName的aware回调
        if (instance instanceof BeanNameAware) {
            ((BeanNameAware)instance).setBeanName(beanName);
        }
    ...
}
```

service类要实现BeanNameAware，才能进行aware回调

## Spring的初始化

创建接口InitializeBean，需要service类实现该接口
```
public interface InitializeBean {  
    void afterPropertiesSet();  
}
```

MyApplicationContext中的createBean方法写入
```
 
public Object createBean(String beanName,BeanDefinition beanDefinition) {
    ...
        //类的初始化  
		if (instance instanceof InitializeBean) {  
			((InitializeBean)instance).afterPropertiesSet();  
		} 
    ...
}
  
```
## BeanPostProcessor机制

创建接口BeanPostProcessor

```
public interface BeanPostProcessor {  
    void postProcessorBeforeInitialization(String beanName,Object bean);  
    void postProcessorAfterInitialization(String beanName,Object bean);  
}
```

创建自定义的MyBeanPostProcessor类实现BeanPostProcessor接口及其方法，在其中对service类实现操作，并将MyBeanPostProcessor加入Spring容器（@Component）
```
@Component  
public class MyBeanPostProcessor implements BeanPostProcessor {  
    @Override  
    public void postProcessorBeforeInitialization(String beanName, Object bean) {  
    //根据beanName名字判断/根据bean类型判断
    //操作
    }  
  
    @Override  
    public void postProcessorAfterInitialization(String beanName, Object bean) {  
  
    }  
}
```

创建BeanPostProcessor的list集合以放入实现该接口的实例。
扫描类注解时加入（在MyApplicationContext的构造方法）
```
...
private ArrayList<BeanPostProcessor> beanPostProcessors = new ArrayList<>();
...
```

```
if (clazz.isAnnotationPresent(Component.class)) {  
    if (BeanPostProcessor.class.isAssignableFrom(clazz)) {  //该类实现接口
        BeanPostProcessor instance = (BeanPostProcessor) clazz.newInstance();  
        beanPostProcessors.add(instance);  
  
    }  
  
    String beanName = clazz.getAnnotation(Component.class).value();  
    if ("".equals(beanName)) {  
        beanName = Introspector.decapitalize(clazz.getSimpleName());  
    }
    ...
```

在createBean方法类初始化前后实现BeanPostProcessor实现类的自定义方法中的操作

```
...
//beanName的aware回调  
if (instance instanceof BeanNameAware) {  
    ((BeanNameAware)instance).setBeanName(beanName);  
}  
  
//实现BeanPostProcessor方法  
for (BeanPostProcessor beanPostProcessor : beanPostProcessors) {  
    beanPostProcessor.postProcessorBeforeInitialization(beanName,instance);  
}  
  
//类的初始化  
if (instance instanceof InitializeBean) {  
    ((InitializeBean)instance).afterPropertiesSet();  
}  
  
//实现BeanPostProcessor方法  
for (BeanPostProcessor beanPostProcessor : beanPostProcessors) {  
    beanPostProcessor.postProcessorAfterInitialization(beanName,instance);  
}
...
```

## AOP机制

现在createBean方法return的是一个普通对象，而AOP要求返回的对象是一个代理对象

使用jdk代理（service类需要接口）
目标：createBean方法经过BeanPostProcessor实现类的方法后返回代理对象

修改接口BeanPostProcessor返回值

```
public interface BeanPostProcessor {  
    Object postProcessorBeforeInitialization(String beanName,Object bean);  
    Object postProcessorAfterInitialization(String beanName,Object bean);  
}
```

修改MyBeanPostProcessor类
代理逻辑中实现切面方法
```
@Component  
public class MyBeanPostProcessor implements BeanPostProcessor {  
    @Override  
    public Object postProcessorBeforeInitialization(String beanName, Object bean) {  
        return bean;  
    }  
  
    @Override  
    public Object postProcessorAfterInitialization(String beanName, Object bean) {  
        if (beanName.equals("userService")) {  
            Object proxyInstance = Proxy.newProxyInstance(MyBeanPostProcessor.class.getClassLoader(), bean.getClass().getInterfaces(), new InvocationHandler() {  
                @Override  
                public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
                    //切面逻辑  
					return method.invoke(bean,args);  
                }  
            });  
  
            return proxyInstance;  //return一个代理对象
        }  
        return bean;  
    }  
}
```

修改createBean方法
```
//beanName的aware回调  
if (instance instanceof BeanNameAware) {  
    ((BeanNameAware)instance).setBeanName(beanName);  
}  
  
//实现BeanPostProcessor方法  
for (BeanPostProcessor beanPostProcessor : beanPostProcessors) {  
    instance = beanPostProcessor.postProcessorBeforeInitialization(beanName,instance);  
}  
  
//类的初始化  
if (instance instanceof InitializeBean) {  
    ((InitializeBean)instance).afterPropertiesSet();  
}  
  
//实现BeanPostProcessor方法  
for (BeanPostProcessor beanPostProcessor : beanPostProcessors) {  
    instance = beanPostProcessor.postProcessorAfterInitialization(beanName,instance);  
}  
  
return instance;
```
如果经过BeanPostProcessor方法，此时return的instance就是我们自定义的代理对象

## Bean的生命周期

#### bean创建的生命周期
> service.class 
    -> 推断构造方法
	-> 对象 
	-> DI 
	-> 初始化前（@PostConstruct）
	-> 初始化 （afterPropertiesSet方法）
	-> 初始化后 （AOP）
	-> （代理对象）
	-> 放入单例池 
	-> bean对象

#### 初始化前和初始化时

假设service类中有一个属性private User admin，如何给在bean创建的时候给这个属性赋值？
如果利用Autowired，spring并不知道该给admin赋值为什么（比如查询数据库得到赋值），所以要自定义方法来给admin赋值。

>实现1:
	此时可以写一个方法f，在f中实现admin的赋值逻辑，并在初始化前进行f的调用。
	如何进行初始化前调用？在方法上使用@PostConstruct注解

	逻辑如下：
```
for(Method method: Service. getClass). getDeclaredMethods()) {
		if (method.isAnnotationPresent(PostConstruct.class)) {
			method.invoke(userService1,null);
	}
```

>实现2:
	service类实现InitializingBean接口，重写其中的afterPropertiesSet方法，在方法中实现逻辑赋值
	该方法会在spring初始化的时候被调用

>  该方法的实现逻辑：
>  进入AbstractAutowiredCapableBeanFactory类的doCreateBean方法，
>  用构造方法得到对象实例，
>  解决循环依赖问题，
>  populatedBean方法属性填充，
>  initializeBean方法 {
> 	 aware回调，
> 	 初始化前，
> 	 初始化：判断有没有实现InitializingBean，执行afterPropertiesSet方法，
> 	 初始化后
>  }，
>  ......
>  

#### 推断构造方法

推断构造方法的底层原理？

>  Q1：private OrderService orderService并未加@Autowired注解。如果有1，输出什么？如果有1、2，输出什么？如果有2、3，输出什么？
```
@Component
public class UserService {
	private OrderService orderService;
	1️⃣public UserService() {
		System.out.println(0);
	｝
	2️⃣public UserService(OrderService orderService) {
		this.orderService = orderService;
		System.out.println(1);
	}
	3️⃣public UserService(OrderService orderService, OrderService orderService1) {
		this.orderService = orderService;
		System.out.println(2);
	}
	...
```
>	结果：
>		1.输出0，并且属性orderService为null。
>		2.输出1，并且属性orderService不为空。
>		3.报错

>	原因分析：
>		如过仅存在一个构造方法，则spring会调用该构造方法；
>		如果存在多个构造方法，则spring会在这些构造方法中寻找无参构造方法。如果找到就会调用，找不到就会报错。

>	Q2：如何让spring调用指定的构造方法？
>		在想被调用的构造方法加上@Autowired注解

>	Q3：前面结果2属性orderService不为null，spring如何为属性赋值？
>		属性如果不是bean，则会报错；
>		属性如果是单例bean，就会去单例池里寻找，如果有就会赋值，没有会创建bean；
>		属性如果不是单例bean，就会直接创建bean

>	Q4：在map<beanName,Bean对象>单例池中寻找bean的时候，如果根据beanName查找不可行，因为public UserService(OrderService orderService)中形参名字可以随便取。所以根据类型来取得。但是根据类型获取会出现一个value对应多个key的情况，或者多个value都是同一类型。如何解决？
```
@Component
public class OrderService {}


@Bean
public OrderService orderService1(){return new OrderService();}
@Bean
public OrderService orderService2(){return new OrderService();}

//这三个orderService不是一个对象，但是类型相同（即多个value都是同一类型）
```
>		解决方法：先根据type查找。如果type查到有多个，就根据name筛选；如果只查到一个，就直接赋值。

>因此，推断构造方法有两步：
>	1.选择构造方法
>	2.实现构造方法入参属性赋值

#### 依赖注入

依赖注入的原理？

>	1.查找属性
>	2.与构造方法入参相同的方法进行属性赋值

#### 代理对象

Q1：放入单例池的是原对象还是代理对象？

> 如果需要实现AOP，则放入的是代理对象；
> 如果不需要，则放入的是原对象

Q2：代理对象被标记@Autowired的属性的值是否为空？

>	属性值为空，因为代理对象产生后并未进行DI就加入了单例池。

Q3：如果为空，那为什么打印输出该属性的时候并不为空？
>	代理类创建时有一个targrt属性，并将代理类的普通类赋值给该target属性。调用test方法时，执行的是代理类的test方法。代理类的test方法先进行AOP，然后调用target.test()，即调用普通类的test方法。target是普通类进行了DI，因此避免了属性为空的问题。
```
class UserServiceProxy extends UserService {
	UserService target;
	public void test(){
		//切面逻辑@Before
		//target.test();//普通对象.test()
	}
}
```

Q4：Q3为什么通过建立target属性来解决而不是直接调用super.test()？
>	？？我也不清楚

Q5：代理类可以不继承普通类吗？（cglib代理）
>	不可以。不继承就不能将代理类强制转换为普通类了。
```
UserService userService = (UserService) applicationContext.getBean("userService")
```

## 事务管理

事务切面逻辑：

>	添加@Transaction注解开启事务
>	-> *事务管理器* 新建数据库连接connection
>	-> connection.autocommit = false
>	-> target.test()
>	-> connection.commit()/rollback()

Q1：为什么spring会让事务管理器建立连接，而不直接让jdbcTemplate建立连接？
>	因为jdbcTemplate建立的连接默认autocommit = true，会直接提交，起不到事务管理的作用

Q2：在service类中添加如下方法，调用test方法，a方法的@Transactional注解会不会起作用？
```
@Transactional
public void test() {
	jdbcTemplate.execute("...");
	a();
}

@Transactional(propagation = Propagation.NEVER)
public void a {
	jdbcTemplate.execute("...");
}
...

```

>	不会。
>	-> 调用service.test()
>	-> 进入代理类的test方法
>	-> 代理了执行target.test()
>	-> 由于target为普通service类，@Transactional注解不起作用
>	-> 不会抛出异常（正常情况下@Transactional(propagation = Propagation.NEVER)会导致抛出异常）
>	
>	为什么test方法的@Transactional注解有用？
>	因为调用service.test()时执行方法的是代理类，会去处理@Transactional注解。而执行a方法的时候只是普通类。

Q3：如何解决Q2的问题？

>	1.拆分类：拆分出另外一个类s1，在s1中写a方法，再将s类注入原service(s)类，在test方法中调用s.a()即可
>	2.自己注入自己
>	3.其他方法拿到当前类代理对象，例如AopContext.currentProxy()

```
@Autowired
private Service s1;

@Transactional
public void test() {
	jdbcTemplate.execute("...");
	s1.a();
}

=================================

@Autowired
private Service s;

@Transactional
public void test() {
	jdbcTemplate.execute("...");
	s.a();
}
```

Q4：@Configuration注解对事务管理的影响（config类不加@Configuration注解错误问题）

>	我们知道，
>	1️⃣ spring的事务管理器创建connection放在LocalThread<Map<Datasource,connection>>中。
>		*“为什么泛形里面放的是map而不直接是connection?
>			因为现实工程中可能会使用多种datasource。”*
>	2️⃣ 不加@Configuration注解时，jdbcTemplate和事务管理器创建的datasource对象不是同一个对象，只有事务管理器创建出来的对象才有事务处理的功能，jdbcTemplate的autocommit = true
>	
>	因此，加上@Configuration注解会使jdbcTemplate调用的是事务管理器创建的datasource，而不会自己再创建一个新的datasource，避免事务管理的失效。

Q5：@Configuration注解解决Q4的原理？

>	*@Configuration，AOP，@Lazy都是使用的动态代理机制*
>	AppConfig加上@Configuration注解之后会生成一个动态代理类

```
public classAppConfig {//config类
	@Bean
	public IdbcTemplate jdbcTemplate() {
		return new JdbcTemplate(dataSource());
	}
	@Bean
	public PlatformTransactionManager transactionManager() {
		DataSourceTransactionManager transactionManager = new DataSourceTransactionManager();
		transactionManager.setDataSource(dataSource());
		return transactionManager;
	}
	@Bean
	public void datasource() {
		...
	}
	
	...
```

```
class UserServiceProxy extends AppConfig｛//config代理类
	UserService target;
	public void jdbcTemplate(){
		//代理逻辑
		super.jdbcTemplate();
	}
	public void datasource(){
		//代理逻辑
	}
｝
```

>	*config类中jdbcTemplate()和transactionManager()方法中的datasource()方法都是代理类调用的*

>	代理类来调用jdbcTemplate()和datasource()方法。
>	代理类在调用这两个方法时，会首先执行代理逻辑。即：在执行jdbcTemplate()方法时，先检测spring容器里面有没有datasource，如果有就直接返回spring容器中的datasource；如果没有就调用datasource()方法去创建一个datasource并放入spring容器。
>	通过这种方式来保证jdbcTemplate()和datasource()得到的是同一个datasource

```
//在config类中

@Bean
public JdbcTemplate jdbcTemplate() {
	return new JdbcTemplate (dataSource)());
}
@Bean
public PlatformTransactionManager transactionManager() {
	DataSourceTransactionManager transactionManager = new DataSourceTransactionManager();
	transactionManager.setDataSource(dataSource1());
	return transactionManager;
}
```

>	上述代码的情况：如果jdbcTemplate()和transactionManager()方法中的datasource不是同一个，即使加@Configuration注解也会出现Q4错误


## 循环依赖

>*三级缓存(三个map)
>	singletonobjects(即单例池) -> 保存拥有完整生命周期的bean
>	earlySingletonobjects -> bean被循环依赖，提前产生一个代理对象，保存到二级缓存，保证单例
>	singletonFactories -> 打破循环，先存到三级缓存，出现循环依赖的情况下才能拿到被依赖的对象*

#### 引入

>背景：AService类注入了BService类属性，BService类注入了AService类属性

>解决方法：利用一个map解决循环依赖问题

AService的Bean的生命周期：
1. ﻿﻿实例化--->AService普通对象--->map.put("AService",AService普通对象）
2. ﻿﻿﻿填充bService--->单例池Map--->创建BService
	BService的Bean的生命周期：
	1. ﻿﻿实例化--->普通对象
	2. ﻿﻿填充aService--->单例池Map--->map--->AService普通对象
	3. ﻿﻿填充其他属性
	4. ﻿﻿﻿做一些其他的事情（AOP）---> AService的代理对象
	5. ﻿﻿﻿添加到单例池
3. ﻿﻿填充其他属性
4. ﻿﻿﻿做一些其他的事情
5. ﻿﻿添加到单例池

>出现问题：注入bService的AService属性是AService的普通对象，而加入单例池是AService的代理对象

>解决方法：未出现循环依赖的bean不提前进行AOP，只有出现循环依赖的bean提前进行AOP。加入﻿creatingSet存放正在进行初始化的bean，在第2.2通过检查creatingSet判断循环依赖，决定是否提前进行AOP。

1. ﻿﻿﻿ creatingSet<AService›
2. ﻿﻿实例化-->AService普通对象
3. ﻿﻿﻿填充bService--->单例池Map--->创建BService
	BService的Bean的生命周期：
	2.1 实例化-->普通对象
	2.2 填充aService--->单例池Map--->creatingSet--->AService出现了循环依赖--->*AOP---> aService代理对象--->添加到单例池*
	2.3 填充其他属性
	2.4 做一些其他的事情（AOP）
	2.5 添加到单例池
4. ﻿﻿填充其他属性
*5. ﻿﻿做一些其他的事情（AOP）--->AService代理对象
6. 添加到单例池*
7. ﻿﻿﻿creatingSet.remove<'AService'>

>出现问题：是否应该将5、6提前到2.2斜体部分？

>答案：不应该在2.2斜体部分加入单例池。
		因为存在一些潜在的问题，例如：
		1. 此时aService普通对象还未初始化完成，代理对象创建过程中可能拿不到普通对象
		2. 此时aService普通对象还未初始化完成，因此aService代理对象中的target属性为空或不完整。此时如果有另外一个线程使用单例池中的aService代理对象，就会出现问题。

#### 二级缓存

>出现问题：
>背景：如果aService依赖bService和cService，而bService和cService都依赖aService，那bService和cService在初始化的时候都需要创建aService的代理对象，与单例模式产生矛盾。

>解决方法：利用第二级缓存earlySingletonobjects，在b需要a的时候创建a的代理对象并将其加入earlySingletonobjects，c在需要a的时候直接从二级缓存里面拿。

*二级缓存的作用：存储出现循环依赖问题时还未完成完整生命周期的早期的bean对象，保持单例性*

>问题：什么时候将aService代理对象加入单例池？

>答案：在aService填充完所有属性并进行完其他的事情（如AOP）之后，earlySingletonobjects.get(AService)得到a的代理对象，加入单例池

>问题：第四步“填充其他属性”是aService普通对象还是代理对象？

>答案：普通对象。代理对象存在的意义仅仅是执行切面逻辑。


#### 三级缓存

*三级缓存的作用是打破循环。*

singletonFactories里存放的是(beanName，lambda表达式)：
```
singletonFactories.put('AService',()-> getEarlyBeanReference(beanName,mbd,AService普通对象))
```

源码：
```
//判断是否是单例，是否支持循环依赖（默认true），是否正在被创建
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences && isSingletonCurrentlyInCreation(beanName));

//如果都是
if(earlySingletonExposure) {
	if(logger.isTraceEnabled()) {
		Logger.trace("Eagerly caching bean '" + beanName + "'to allow for resolving potential circular references");
	}
	//加入三级缓存
	addSingletonFactory(beanName,() -> getEarlyBeanReference(beanName, mbd,
bean));
}
```

如果在一二级缓存里面都没找到，就去第三级缓存找，执行()-> getEarlyBeanReference(beanName,mbd,AService普通对象)。
如果该bean需要进行AOP，那执行的结果是进行AOP，创建代理对象。
如果不需要，那执行的结果是返回普通对象。
最后将代理/普通对象放入二级缓存。


a填充b的属性时，会调用getSingleton方法：
```
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	Object singletonobject = this. singletonobjects.get (beanName);
	if(singletonObject == null &d isSingletonCurrentlyInCreation(beanName)) {//当前索要的对象为空且正在创建中，即出现了循环依赖
		singletonObject = this. earlySingletonobjects. get (beanName);
		if (singletonobject == null && allowEarlyReference) {
			synchronized (this. singletonObjects) {
			// Consistent creation of early reference within full singleton lock
				singletonobject = this. singletonobjects.get (beanName);
				if (singletonobject == null) {
					singletonobject = this. earlySingletonobjects.get (beanName);
						if (singletonObject == null) {//二级缓存没有
							ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
								if (singletonFactory != null) {//去三级缓存找
									singletonobject = singletonFactory.getobject;
									this. earlySingletonobjects. put (beanName, singLetonobject);//加入二级缓存中
									this. singletonFactories. remove (beanName);//从三级缓存中移除，即移除lambda表达式
								}
							}
						}
					}
				}
			}
		}
	}
	return singletonObject;
}
```

> 为什么要从三级缓存中移除？

>因为对象是单例的，lambda表达式（()-> getEarlyBeanReference(beanName,mbd,AService普通对象)）只需要执行一次，保证单例。
>同理，当需要去三级缓存中存的时候，二级缓存中也会执行移除
>例如：
```
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
	Assert. notNull(singletonFactory, "Singleton factory must not be null");
	synchronized (this.singletonObjects) {
		if (!this. singletonobjects. containsKey(beanName)) {
			this. singletonFactories put(beanName, singletonFactory);
			this.earlySingleton0bjects.remove(beanName);
			this.registeredSingletons.add (beanName);
		}
	}
}
```

>*5、6步之间还会调用getSingleton方法，从二级缓存中拿到代理对象*


>总结：
>*对象创建时，先存在三级缓存中，在出现循环依赖的时候去三级缓存取，然后执行取到的lambda表达式并执行，进行AOP得到代理对象，然后将代理对象存进二级缓存，然后执行接下来的创建步骤，创建完成后放入一级缓存。*


#### Map:earlyProxyReferences

>如果在2.2提前进行AOP，那原本该进行AOP的时候（第5步）如何进行？

```
@Override
public Object postProcessAfterInitialization (Nullable Object bean, String beanName) {
	if (bean != null) {
		Object cacheKey = getCacheKey (bean. getClass), beanName);
		if (this.earlyProxyReferences.remove(cacheKey) != bean) {
			return wrapIfNecessary(bean, beanName, cacheKey);
		}
	}
	return bean;
｝
@Override
public Object getEarlyBeanReference(Object bean, String beanName) {
	Object cacheKey = getCacheKey(bean. getClass), beanName);
	this.earlyProxyReferences.put(cacheKey, bean);
	return wrapIfNecessary(bean, beanName, cachekey);
}
```

>通过一个map "earlyProxyReferences"来区分bean是否进行过AOP。
>getEarlyBeanReference方法是lambda表达式调用的方法。如果提前进行AOP，该方法被调用，对该bean产生一个cacheKey放入earlyProxyReferences中，并且调用wrapIfNecessary方法。
>postProcessAfterInitialization是正常AOP调用的方法。if的意义在于判断earlyProxyReferences中是否有该bean的cacheKey。如果有，就直接返回*普通对象（此处不必着急返回代理对象，因为5、6步之间还会调用getSingleton方法，从二级缓存中拿到代理对象）*；如果没有，即this.earlyProxyReferences. remove(cacheKey) == null，就调用wrapIfNecessary方法。


#### @Lazy

>问题：如果在a构造方法里（a初始化的时候）加入this.bService = bService，则根本创建不出来a的普通对象，因此spring解决不了循环依赖问题。

```
@Component
public class AService {
	private BService bService;
	
	@Lazy
	public AService(BService bService) {
		this.bService = bService;
	}
	public void test() {
		bService.toString();
	}
	...
```

>解决方法：加上@Lazy注解。加注解后，在创建a的bean的时候，会创建一个b的代理对象注入a，不会出现循环依赖问题。等到真正调用关于b的方法（例如test方法）的时候，b才会真正被创建出来。
>*也可以解决@Async报错问题。