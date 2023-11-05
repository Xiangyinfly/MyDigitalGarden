---
{"dg-publish":true,"permalink":"/spring/spring-security/","dgPassFrontmatter":true}
---


# 概述

## SpringSecurity流程
![SS流程.png|undefined](/img/user/Spring/assets/SS%E6%B5%81%E7%A8%8B.png)
其中的过滤器链
![SS过滤器链.png|undefined](/img/user/Spring/assets/SS%E8%BF%87%E6%BB%A4%E5%99%A8%E9%93%BE.png)



## 登录校验流程![登录校验流程.png|undefined](/img/user/Spring/assets/%E7%99%BB%E5%BD%95%E6%A0%A1%E9%AA%8C%E6%B5%81%E7%A8%8B.png)

## 认证流程![认证流程.png|undefined](/img/user/Spring/assets/%E8%AE%A4%E8%AF%81%E6%B5%81%E7%A8%8B.png)
![认证流程2.png|undefined](/img/user/Spring/assets/%E8%AE%A4%E8%AF%81%E6%B5%81%E7%A8%8B2.png)![认证流程4.png|undefined](/img/user/Spring/assets/%E8%AE%A4%E8%AF%81%E6%B5%81%E7%A8%8B4.png)![认证流程3.png|undefined](/img/user/Spring/assets/%E8%AE%A4%E8%AF%81%E6%B5%81%E7%A8%8B3.png)![认证流程5.png|undefined](/img/user/Spring/assets/%E8%AE%A4%E8%AF%81%E6%B5%81%E7%A8%8B5.png)

自定义登录页面


认证的核心是各种AuthenticationProvider，其中使用的最多的一个DaoAuthenticationProvider。

![Pasted image 20231104221101.png|undefined](/img/user/Spring/assets/Pasted%20image%2020231104221101.png)

UserDetails是用户信息的实体类，UserDetailsService自定义实现，根据用户名密码加载UserDetails，PasswordEncoder就是密码的加密类。
DaoAuthenticationProvider通过UserDetailsService以及PasswordEncoder，将用户名密码替换成UserDetails和Authorities。
DaoAuthenticationProvider的由来：在WebSecurityConfigurerAdapter中，以下代码之前都看过，AuthenticationManager 的由来也都说明过，其中有一个parent被所有的子AuthenticationManager共享，而这个DaoAuthenticationProvider就是包括在parent中。


## 自定义图形验证码

引入依赖
```<dependency>  
    <groupId>com.github.penggle</groupId>  
    <artifactId>kaptcha</artifactId>  
    <version>2.3.2</version>  
</dependency>
```

配置config类
```
@Configuration  
public class CaptchaConfig {  
    @Bean  
    public Producer producer(){  
  
        Properties properties = new Properties();  
        properties.setProperty("kaptcha.image.width","150");  
        properties.setProperty("kaptcha.image.height","50");  
        properties.setProperty("kaptcha.textproducer.char.string","0123456789");  
        properties.setProperty("kaptcha.textproducer.char.length","4");  
  
        Config config = new Config(properties);  
        DefaultKaptcha defaultKaptcha = new DefaultKaptcha();  
        defaultKaptcha.setConfig(config);  
  
        return defaultKaptcha;  
    }  
}
```


配置controller类
```
@Slf4j  
@Controller  
@RequestMapping("/captcha")  
public class CaptchaController {  
  
    @Autowired  
    private Producer producer;  
  
    @SneakyThrows  
    @GetMapping("/captcha.jpg")  
    public void getCaptcha(HttpServletRequest request, HttpServletResponse response){  
        response.setContentType("image/jpeg");  
        String capText = producer.createText();  
        log.info("验证码:{}",capText);  
  
        request.getSession().setAttribute("captcha",capText);  
        BufferedImage image = producer.createImage(capText);  
        ServletOutputStream out = response.getOutputStream();  
  
        ImageIO.write(image,"jpg",out);  
        out.flush();  
  
    }  
  
}
```

配置异常类
```
public class VerificationCodeException extends AuthenticationException {  
  
    public VerificationCodeException(){  
        super("图形验证码验证失败!");  
    }  
}
```


配置校验过滤器
```
@Slf4j  
public class VerificationCodeFilter extends OncePerRequestFilter {  //每次请求都会执行
  
    //验证码异常跳转到/login  重写失败方法
    AuthenticationFailureHandler authenticationFailureHandler = new AuthenticationFailureHandler() {  
        @Override  
        public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {  
            log.info("异常:{}",exception.getMessage());  
            response.sendRedirect("/login");  
        }  
    };  
  
    @Override  
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {  
        if(!("/login".equals(request.getRequestURI()) && request.getMethod().equals("POST"))){//login页面不走验证码  
            filterChain.doFilter(request,response);  
        }else{  
            try {  
                verificationCode(request,response);  
                filterChain.doFilter(request,response);  
            } catch (VerificationCodeException e) {  
                authenticationFailureHandler.onAuthenticationFailure(request,response,e);  
            }  
  
        }  
    }  
  
    public void verificationCode(HttpServletRequest request, HttpServletResponse response) throws VerificationCodeException {  
        String requestCode = request.getParameter("captcha");  
        HttpSession session = request.getSession();  
        String sessionCode = (String) session.getAttribute("captcha");  
        if(!StringUtils.isEmpty(sessionCode)){  
            session.removeAttribute("captcha");}  
  
        if(StringUtils.isEmpty(requestCode) || StringUtils.isEmpty(sessionCode) || !requestCode.equals(sessionCode)){  
            throw new VerificationCodeException();  
        }  
  
    }  
}
```

在SecurityConfig类添加验证码过滤器
```
http.addFilterBefore(new VerificationCodeFilter(),UsernamePasswordAuthenticationFilter.class);
```

前端对接
```
验证码:<input type="text" name="captcha"><br/>  
<img src="/captcha/captcha.jpg" />
```


SpringSecurity提供的另一种校验方式：

1、找到自定义DaoAuthenticationProvider在哪里替换SpringSecurity自带的DaoAuthenticationProvider
@EnableWebSecurity
    HttpSecurityConfiguration
        httpSecurity()
            authenticationManager()
                this.authenticationConfiguration.getAuthenticationManager()
                    this.authenticationManager = authBuilder.build();
                        this.object = doBuild();
                            configure();
                                InitializeAuthenticationProviderBeanManagerConfigurer
                                    //在这里能获取到值说明是我们自定义了一个AuthenticationProvider实现类
                                    AuthenticationProvider authenticationProvider = getBeanOrNull(AuthenticationProvider.class);
                                    
InitializeAuthenticationProviderBeanManagerConfigurer

创建MyDaoAuthenticationProvider
```
@Component  
public class MyDaoAuthenticationProvider extends DaoAuthenticationProvider {  
    public MyDaoAuthenticationProvider(UserDetailsService userDetailsService, PasswordEncoder passwordEncoder) {  
        super. setUserDetailsService(userDetailsService);  
        super. setPasswordEncoder (passwordEncoder);  
    }  
  
    //在这里获取request做比对  
    //authentication中的details就是MyWebAuthenticationDetails  
    @Override  
    protected void additionalAuthenticationChecks(UserDetails userDetails, UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {  
        MyWebAuthenticationDetails details = (MyWebAuthenticationDetails) authentication.getDetails();  
  
        if(!details.isImageCodeIsRight()){  
            throw new VerificationCodeException();  
        }  
  
        super.additionalAuthenticationChecks(userDetails, authentication);  
    }  
}
```

创建MyWebAuthenticationDetails
```
public class MyWebAuthenticationDetails extends WebAuthenticationDetails {  
  
    //加一个标识判断验证码是否正确  
    private boolean imageCodeIsRight;  
  
    public boolean isImageCodeIsRight() {  
        return imageCodeIsRight;  
    }  
  
    public MyWebAuthenticationDetails(HttpServletRequest request) {  
        super(request);  
        String requestCode = request.getParameter("captcha");  
        HttpSession session = request.getSession();  
        String sessionCode = (String) session.getAttribute("captcha");  
        if(!StringUtils.isEmpty(sessionCode)){  
            session.removeAttribute("captcha");  
            if(sessionCode.equals(requestCode)){  
                this.imageCodeIsRight = true;  
            }  
        }  
    }  
  
    public MyWebAuthenticationDetails(String remoteAddress, String sessionId) {  
        super(remoteAddress, sessionId);  
    }  
}
```

创建MyWebAuthenticationDetailsSource
```
public class MyWebAuthenticationDetailsSource extends WebAuthenticationDetailsSource {  
    @Override  
    public WebAuthenticationDetails buildDetails(HttpServletRequest context) {  
        return super.buildDetails(context);  
    }  
}
```

config类中加入
```
http.formLogin(formLogin -> formLogin  
        .authenticationDetailsSource(new MyWebAuthenticationDetailsSource())
```

2、找到自定义的DaoAuthenticationProvider中如何使用到request
@EnableWebSecurity SecurityConfig
    configure(B http)
        //这里如果我们自己添加一个自定义的authenticationDetailsSource 会将默认的替换掉
        if (this.authenticationDetailsSource != null) {
            this.authFilter.setAuthenticationDetailsSource(this.authenticationDetailsSource);
        }





持久化令牌
将登录信息记录到数据库中【token令牌:和username关联起来的一个值】
SpringSecurity默认自带了一个令牌管理的Dao 但是是使用的spring-jdbc:JdbcTokenRepositoryImpl
改造成mybatis-plus的

建立登录信息表

建立entity类SysToken
```
@TableName ("t_sys_token" )
@Data
public class SysToken {
	@TableField (value = "username")
	private String username;
	@TableField (value = "series")
	private String series;
	@TableField(value = "token")
	private String tokenValue;
	@TableField (value = "last_used" )
	private Date date;
}
```

建立SysTokenMapper
建立SysPersistentTokenRepositoryImpl
```
public class SysPersistentTokenRepositoryImpl implements PersistentTokenRepository {

}

```

