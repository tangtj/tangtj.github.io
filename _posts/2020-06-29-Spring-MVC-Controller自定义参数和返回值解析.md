---
layout: post
title: "Spring MVC Controller自定义参数和返回值解析"
category: 工作
tags: 
  - spring
  - java
typora-root-url: ..
---

本文代码示例 ：[Github](https://github.com/tangtj1/BlogExamlpe/tree/master/src/main/java/cn/tangtj/blogexample/demo2)

### 需求

使用Spring mvc作为Web框架，有时候api请求进来的时候需要一些信息，比如请求的IP、用户的信息等等。常规做法是比如获取ip

```java
  		@RequestMapping(value="/index.html",method=RequestMethod.GET)
        public String index(HttpServletRequest req) {
            String remoteAddr = req.getRemoteAddr();
            System.out.println(remoteAddr);
            return "index";
       }
```

这样每次获取IP都十分繁琐。

翻看SpringMVC文档。框架支持十几种默认的参数注入，例如HttpServletRequest、HttpServletResponse、InputStream等等。既然框架支持以上参数的默认注入，那么我们也应该可以做到支持自己的自定义类型解析注入到方法参数中。



### 源码分析

`DispatcherServlet#doService`是所有请求的入口，从这里一步一步看。

在`InvocableHandlerMethod#getMethodArgumentValues`

![cead3e00a0a8fa975044f37808eb4c0](/file/image/cead3e00a0a8fa975044f37808eb4c0.png)

可以看到，获取Controller方法入参的信息，并挨个判断是否支持解析。不支持的直接会抛出异常，支持的则会接着进行下去对参数进行处理。

查看一下这个类`HandlerMethodArgumentResolverComposite`他集成了`HandlerMethodArgumentResolver`接口，使用了代理模式，内部存有20来个`HandlerMethodArgumentResolver`实现类，由它统一对外进行判断内部调用对应的实现接口。

![TIM截图20200629171105](/file/image/TIM截图20200629171105.png)

![TIM截图20200629171127](/file/image/TIM截图20200629171127.png)

`HandlerMethodArgumentResolver`接口

![微信图片_20200629171755](/file/image/微信图片_20200629171755.png)

接口中有两个方法。一个判断是否支持该参数，另一个就是参数对应的对象。

根据注释来说的话，这是符合需求的接口。并且可以看到`HandlerMethodArgumentResolverComposite#addResolve`方法允许添加解析器。

现在思路有了。实现`HandlerMethodArgumentResolver`接口，并配置到spring中。



### 实现

写一个demo，获取Request IP注入到入参中。

实体类：

```
public class RequestInfo {

    private String requestIp;

    public String getRequestIp() {
        return requestIp;
    }

    public void setRequestIp(String requestIp) {
        this.requestIp = requestIp;
    }
}
```



`HandlerMethodArgumentResolver`实现类：

```java
public class RequestIpArgResolver implements HandlerMethodArgumentResolver {
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return RequestInfo.class.isAssignableFrom(parameter.getParameterType());
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        HttpServletRequest  req = webRequest.getNativeRequest(HttpServletRequest.class);
        if(req == null){
            return null;
        }
        String addr = req.getRemoteAddr();
        RequestInfo info = new RequestInfo();
        info.setRequestIp(addr);
        return info;
    }
}
```



根据文档说明，加入到Web配置中：

```
@Configuration
public class Web2Config implements WebMvcConfigurer {

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new RequestIpArgResolver());
    }
}
```



Controller

```
@RestController
@RequestMapping("/ip")
public class ReqIPController {

    @GetMapping
    public Object req(RequestInfo requestInfo){
        return requestInfo;
    }
}
```



运行结果图：

![TIM截图20200629183201](/file/image/TIM截图20200629183201.png)



获取到了请求的IP，现在的话只需要在方法参数中加入实体类，就可以获取到IP。