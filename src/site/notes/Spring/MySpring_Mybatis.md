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