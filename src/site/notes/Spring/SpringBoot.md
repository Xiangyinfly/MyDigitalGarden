---
{"dg-publish":true,"permalink":"/spring/spring-boot/","dgPassFrontmatter":true}
---


# 实现请求的处理

```Java
//接收到的请求交给DispatcherServlet去处理，而DispatcherServlet会找到请求的方法并执行  
tomcat.addServlet(contextPath,"dispatcher",new DispatcherServlet(applicationContext));  
context.addServletMappingDecoded("/*","dispatcher");
```

>为什么在new DispatcherServlet的时候要传入一个applicationContext（即spring容器）？
> 
>因为DispatcherServlet要去@Controller注解的bean中去寻找请求的方法，所以要传入spring容器来寻找bean

在MySpringApplication类的run方法中创建spring容器

```Java
/**  
 * * @param primarySource 配置类  
 */  
public static void run(Class<?> primarySource) {  
    //创建spring容器  
    AnnotationConfigWebApplicationContext applicationContext = new AnnotationConfigWebApplicationContext();  
    //会解析传入类的注解  
    applicationContext.register(primarySource);  
    applicationContext.refresh();  
  
    //启动tomcat，传入AnnotationConfigWebApplicationContext
    startTomcat(applicationContext);  
}
```

其中primarySource为我们传入的配置类
注册过程中会解析配置类的注解，即我们所写的@MySpringBootApplication

在SpringBoot的@SpringBootApplication注解中加入了@ComponentScan注解，
如果该注解没有指定扫描的包路径，默认扫描被解析的类所在的包

**所以SpringBoot默认扫描的包路径是调用run方法传入的类所在包的包路径，而不能简单理解为启动类所在包的包路径**

---

# MySpringApplication类

```Java
public class MySpringApplication {  
    /**  
     *     * @param primarySource 配置类  
     */  
    public static void run(Class<?> primarySource) {  
        //创建spring容器  
        AnnotationConfigWebApplicationContext applicationContext = new AnnotationConfigWebApplicationContext();  
        //会解析传入类的注解  
        applicationContext.register(primarySource);  
        applicationContext.refresh();  
  
        //启动tomcat  
        startTomcat(applicationContext);  
        
    public static void startTomcat(WebApplicationContext applicationContext){  
  
        Tomcat tomcat = new Tomcat();  
  
        Server server = tomcat.getServer();  
        Service service = server.findService("Tomcat");  
  
        Connector connector = new Connector();  
        connector.setPort(8081);  
  
        Engine engine = new StandardEngine();  
        engine.setDefaultHost("localhost");  
  
        Host host = new StandardHost();  
        host.setName("localhost");  
  
        String contextPath = "";  
        Context context = new StandardContext();  
        context.setPath(contextPath);  
        context.addLifecycleListener(new Tomcat.FixContextListener());  
  
        host.addChild(context);  
        engine.addChild(host);  
  
        service.setContainer(engine);  
        service.addConnector(connector);  
  
        //接收到的请求交给DispatcherServlet去处理，而DispatcherServlet会找到请求的方法并执行  
        tomcat.addServlet(contextPath,"dispatcher",new DispatcherServlet(applicationContext));  
        context.addServletMappingDecoded("/*","dispatcher");  
  
        try {  
            tomcat.start();  
        } catch (LifecycleException e) {  
            e.printStackTrace();  
        }  
    }}
```

# @MySpringBootApplication注解类
```Java
@Target({ElementType.TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Inherited  
//如果没有指定扫描的包路径，默认扫描被解析的类所在的包  
@ComponentScan  
  
public @interface MySpringBootApplication {  
}
```

---


# 实现Tomcat和Jetty的切换

>在MySpringApplication类中我们直接指定了Tomcat的启动，这样就无法使用Jetty，如何解决？

新建一个接口WebServer
```Java
public interface WebServer {  
    public void start();  
}
```

写两个类TomcatWebServer和JettyWebServer实现这个接口和其中的start方法

写一个getWebServer方法，在run方法中调用得到我们所使用的webServer并启动。

getWebServer方法通过applicationContext拿到我们存在的WebServer。
因为使用一个webServer，所以得到的webServer要判断唯一性
```Java
public static void run(Class<?> primarySource) {  
    //创建spring容器  
    AnnotationConfigWebApplicationContext applicationContext = new AnnotationConfigWebApplicationContext();  
    //会解析传入类的注解  
    applicationContext.register(primarySource);  
    applicationContext.refresh();  
  
    //启动tomcat  
    //startTomcat(applicationContext);  
    //实现启动WebServer  
    WebServer webServer = getWebServer(applicationContext);  
    webServer.start();  
}  
  
private static WebServer getWebServer(ApplicationContext applicationContext) {  
    Map<String, WebServer> webServers = applicationContext.getBeansOfType(WebServer.class);  
    if (webServers.isEmpty()) {  
        throw new NullPointerException();  
    }    if (webServers.size() > 1) {  
        throw new IllegalStateException();  
    }    //返回唯一的一个  
    return webServers.values().stream().findFirst().get();  
}
```

>如何保证得到的WebServer唯一？
>写一个WebServerAutoConfiguration类，通过@Condition注解实现只将我们使用的WebServer的bean添加到spring中，这样getWebServer得到的WebServer唯一。
```Java
@Configuration  
public class WebServerAutoConfiguration {  
    @Bean  
    @Conditional(JettyCondition.class)  
    public JettyWebServer jettyWebServer() {  
        return new JettyWebServer();  
    }   
    @Bean  
    @Conditional(TomcatCondition.class)  
    public TomcatWebServer tomcatWebServer() {  
        return new TomcatWebServer();  
    }}
```

>@Condition注解如何实现条件？
>以TomcatCondition为例：
```Java
public class TomcatCondition implements Condition {  
    @Override  
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {  
        //通过判断项目里面有没有tomcat的特定类来判断有没有引入tomcat依赖  
        try {  
            //通过类加载器加载tomcat类，如果加载不成功返回false  
            context.getClassLoader().loadClass("org.apache.catalina.startup.Tomcat");  
        } catch (ClassNotFoundException e) {  
            return false;  
        }        return true;  
    }}
```

>SpringBoot依赖中同时含有Tomcat和Jetty依赖，那我们现在的配置两个condition都会满足，还是会有两个WebServer加入spring容器，如何解决？
>在SpringBoot的pom文件中，使tomcat或jetty其中一个的依赖optional属性为true，这样设置的为默认使用optional不为true的那一个；
>在依赖SpringBoot的项目的pom文件中，可以排除不需要使用的一项依赖；如果不排除，就会使用默认的那一个；如果使用的不是默认的，需要加上其依赖

>问题：由于spring扫描的是使用SpringBoot的项目下的包，在我们写的SpringBoot的WebServerAutoConfiguration类不会被扫描到，因此报错，如何解决？
>在@MySpringBootApplication注解类中加入@Import(WebServerAutoConfiguration.class)，引入WebServerAutoConfiguration类


# @ConditionOnClass的实现

```Java
@Target({ElementType.TYPE, ElementType.METHOD})  
@Retention(RetentionPolicy.RUNTIME)  
@Conditional(MyOnClassCondition.class)  
public @interface MyConditionOnClass {  
    String value();  
}
```

其中MyOnClassCondition类来替代之前TomcatCondition类的功能
```Java
public class MyOnClassCondition implements Condition {  
    @Override  
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {  
        try {  
	        //metadata得到注解的value值，进而得到传入的包名
            Map<String, Object> annotationAttributes = metadata.getAnnotationAttributes(MyConditionOnClass.class.getName());  
            String value = (String) annotationAttributes.get("value");  
            context.getClassLoader().loadClass(value);  
        } catch (ClassNotFoundException e) {  
            return false;  
        }        return true;  
    }}
```

在配置类中使用
```Java
@Bean  
//@Conditional(TomcatCondition.class)  
//传入要判断的包名
@MyConditionOnClass("org.apache.catalina.startup.Tomcat")  
public TomcatWebServer tomcatWebServer() {  
    return new TomcatWebServer();  
}
```

# 关于自动配置类

SpringBoot中有很多配置类

## 实现配置类的批量导入

在@SpringBootApplication中含有注解@EnableAutoConfiguration
@EnableAutoConfiguration中含有注解@Import({AutoConfigurationImportSelector.class})
AutoConfigurationImportSelector实现了ImportSelector接口，重写了`String[] selectImports(AnnotationMetadata importingClassMetadata)`方法，此方法得到需要导入的配置类并返回一个string数组

```Java
public String[] selectImports(AnnotationMetadata annotationMetadata) {  
    if (!this.isEnabled(annotationMetadata)) {  
        return NO_IMPORTS;  
    } else {  
        AutoConfigurationEntry autoConfigurationEntry = this.getAutoConfigurationEntry(annotationMetadata);  
        return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());  
    }}


protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {  
    if (!this.isEnabled(annotationMetadata)) {  
        return EMPTY_ENTRY;  
    } else {  
        AnnotationAttributes attributes = this.getAttributes(annotationMetadata);  
        List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);  
        configurations = this.removeDuplicates(configurations);  
        Set<String> exclusions = this.getExclusions(annotationMetadata, attributes);  
        this.checkExcludedClasses(configurations, exclusions);  
        configurations.removeAll(exclusions);  
        configurations = this.getConfigurationClassFilter().filter(configurations);  
        this.fireAutoConfigurationImportEvents(configurations, exclusions);  
        return new AutoConfigurationEntry(configurations, exclusions);  
    }}
```

## 如何得到配置类

jar包中的META-INFO下存在spring.factories文件，其中的k-v定义了jar包中的配置类，selectImports方法会得到其中的k-v（最终通过classLoader.getResources）

`this.getConfigurationClassFilter().filter(configurations); `实现配置类的过滤，只留下我们所使用到的

### 过滤原理

>每个配置类基本上都会有@ConditionOnClass注解，在编译jar包的时候会解析类上的注解，将其写入spring-autoconfigure-metadata.properties文件中。因为解析@ConditionOnClass，所以其中记录了依赖这些配置类必然会存在的一些特征类。过滤器会根据得到的配置类的数组，去spring-autoconfigure-metadata.properties文件中寻找这些配置类，然后判断对应的特征类是否存在，不存在说明项目不使用该配置类配置的依赖，将其过滤掉

### 编译的时候如何实现将解析后的数据写入文件？

>类AbstractProcessor可以实现，其中的process方法定义了数据写入的逻辑。
>AutoConfigureAnnotationProcessor继承AbstractProcessor，专门实现编译时自动配置类的解析和写入