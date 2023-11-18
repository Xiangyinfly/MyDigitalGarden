---
{"dg-publish":true,"permalink":"/spring/spring-security-04/","dgPassFrontmatter":true}
---

# 关于授权的过滤器

- AuthorizationFilter：基于类的授权
- FilterSecurityInterceptor：基于表达式的授权

# AuthorizationFilter及相关类

* AuthorizationFilter

* AuthorizationManager

* AuthorizationDecision

* AuthorizeHttpRequestsConfigurer

* RequestAuthorizationContext

* AuthorizationManagerRequestMatcherRegistry

* AbstractRequestMatcherRegistry

* AuthorizedUrl

## AuthorizationFilter

### doFilter方法

```Java
@Override  
public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain chain)  
       throws ServletException, IOException {  
  
    HttpServletRequest request = (HttpServletRequest) servletRequest;  
    HttpServletResponse response = (HttpServletResponse) servletResponse;  
  
    if (this.observeOncePerRequest && isApplied(request)) {  
       chain.doFilter(request, response);  
       return;  
    }  
  
    if (skipDispatch(request)) {  
       chain.doFilter(request, response);  
       return;  
    }  
  
    String alreadyFilteredAttributeName = getAlreadyFilteredAttributeName();  
    request.setAttribute(alreadyFilteredAttributeName, Boolean.TRUE);  
    try {  
	    //得到授权结果
       AuthorizationDecision decision = this.authorizationManager.check(this::getAuthentication, request);  
       this.eventPublisher.publishAuthorizationEvent(this::getAuthentication, request, decision);  
       if (decision != null && !decision.isGranted()) {  
          throw new AccessDeniedException("Access Denied");  
       }  
       chain.doFilter(request, response);  
    }  
    finally {  
       request.removeAttribute(alreadyFilteredAttributeName);  
    }  
}
```

利用[[Spring/Spring_Security_04#AuthorizationManager\|authorizationManager]]的check方法校验

### getAuthentication方法
```Java
private Authentication getAuthentication() {  
    Authentication authentication = this.securityContextHolderStrategy.getContext().getAuthentication();  
    if (authentication == null) {  
       throw new AuthenticationCredentialsNotFoundException(  
             "An Authentication object was not found in the SecurityContext");  
    }  
    return authentication;  
}
```
## AuthorizationManager

```
//决策能不能访问T类型的对象
An Authorization manager which can determine if an Authentication has access to a specific object.
Type parameters:
<T> – the type of object that the authorization check is being done on.
```

```Java
@FunctionalInterface  
public interface AuthorizationManager<T> {  
  
    default void verify(Supplier<Authentication> authentication, T object) {  
       AuthorizationDecision decision = check(authentication, object);  
       if (decision != null && !decision.isGranted()) {  
          throw new AccessDeniedException("Access Denied");  
       }  
    }  
  
    @Nullable  
    AuthorizationDecision check(Supplier<Authentication> authentication, T object);  
}
```

### check方法

Supplier<\Authentication> authentication：
提供器，可以得到一个authentication，
在[[Spring/Spring_Security_04#AuthorizationFilter\|AuthorizationFilter]]类的[[Spring/Spring_Security_04#doFilter方法\|doFilter]]方法中被调用：this.authorizationManager.check(this::getAuthentication, request)
因此改authentication为[[Spring/Spring_Security_04#AuthorizationFilter\|AuthorizationFilter]]中[[Spring/Spring_Security_04#getAuthentication方法\|getAuthentication]]方法得到的Authentication

