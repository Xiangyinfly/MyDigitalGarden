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

事务切面逻辑？

>	添加@Transaction注解开启事务
>	-> *事务管理器* 新建数据库连接connection
>	-> connection.autocommit = false
>	-> target.test()
>	-> connection.commit()/rollback()
>	-> 

Q1：为什么spring会让事务管理器建立连接，而不直接让jdbcTemplate建立连接？
>	因为jdbcTemplate建立的连接默认autocommit = true，会直接提交，起不到事务管理的作用
