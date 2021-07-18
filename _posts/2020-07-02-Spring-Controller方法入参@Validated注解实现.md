---
layout: post
title: "Spring Controller方法入参@Validated注解实现"
category: 工作
tags: 
  - spring
  - java
typora-root-url: ..
---

**本文代码示例：[GITHUB](https://github.com/tangtj1/BlogExamlpe/tree/master/src/main/java/cn/tangtj/blogexample/demo3)**

### 介绍

@Validated 注解是Spring提供的JSR-303实现的变种注解。

根据注解源码注释他可以用在

1. Controller层方法参数
2. 标注了@Validated注解类的方法

本文主要讲 @Validated 验证 Controller 方法参数实现。

本文所用SpringBoot版本 `2.3.1.RELEASE`

首先使用校验的话需要引入对应的 starter

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

写一个简单的demo:

需要校验的实体类

```
@Data
public class ValidInput {

    @NotNull
    @Min(value = 1,message = "最小不能小于1")
    private Integer value;
}
```

简单的Controller类

```
@RestController
@RequestMapping("/valid")
public class ValidController {

    @PostMapping("/valid")
    public Object valid(@Validated @RequestBody ValidInput input){
        return input;
    }
}
```

打上断点，我们找一下Spring是在哪里对Controller方法入参进行数据校验的。

进行测试的Http请求参数

```
POST http://{{url}}/valid/valid
Content-Type: application/json

{
  "value": -1
}
```

我将value参数设置为-1，实体类对value的限制是最小为1，所以在这里的话会校验失败，返回异常。

请求了一次之后我再控制台日志中看到了这一句。

```
2020-07-02 00:00:08.669  WARN 7632 --- [nio-8081-exec-1] .w.s.m.s.DefaultHandlerExceptionResolver : Resolved [org.springframework.web.bind.MethodArgumentNotValidException: Validation failed for argument [0] in public java.lang.Object cn.tangtj.blogexample.demo3.web.ValidController.valid(cn.tangtj.blogexample.demo3.domain.ValidInput): [Field error in object 'validInput' on field 'value': rejected value [-1]; codes [Min.validInput.value,Min.value,Min.java.lang.Integer,Min]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [validInput.value,value]; arguments []; default message [value],1]; default message [最小不能小于1]] ]

```

确实有发生异常，被框架抓住了并解析了。

### 源码

进入日志中所说的`DefaultHandlerExceptionResolver`类，在`doResolveException`方法上打上了断点。再次行了一次http请求，捕捉到了异常，查看异常的栈信息。

![20200701-validated-01](/file/image/20200701-validated-01.png)

从图中我们可以看出 抛出的异常名为`MethodArgumentNotValidException`，并定位到了是从`RequestResponseBodyMethodProcessor#resolveArgument`方法中抛出来的。

定位到源码如下：

![20200701-validated-02](/file/image/20200701-validated-02.png)

可以看出先是在第一个红箭头处，进行了数据验证，然后判断是否有数据校验错误，如有则抛出

`MethodArgumentNotValidException`异常。

这里的代码看起来十分简单，回到这个类进行分析。

查看这个类的UML结构图

![20200701-validated-03](/file/image/20200701-validated-03.png)

`RequestResponseBodyMethodProcessor` 继承了`AbstractMessageConverterMethodProcessor`并最终实现`HandlerMethodArgumentResolver`接口的`resolveArgument` 方法，并支持验证所有被`@RequestBody`标注的参数。

验证过程就是:

1. 使用对应 HttpMessageConverter 接口将参数转化为实体类。
2. 判断参数上是否有`@Validated`注解，如果有就进行数据校验并将信息存到`bindata`上
3. 判断有无校验异常，有的话就抛出异常。

### 扩展

在这里我想到一个问题，Controller自定义方法参数转换 也是实现的`HandlerMethodArgumentResolver`接口，查看之前的博文 ：[Spring MVC Controller自定义方法参数]({% link _posts/2020-06-29-Spring-MVC-Controller自定义参数和返回值解析.md %})

发现：**如果不加入@Validated校验的相关代码，那么自定义方法参数解析无法使用@Validated进行数据校验**，因为每种类型对应了一个`HandlerMethodArgumentResolver`实现，@Validated校验只对带有@RequestBody有用。

解决方案：按照`RequestResponseBodyMethodProcessor` 的实现照猫画虎加入相关的校验代码。

具体解析实现：

```java
@Component
public class RequestTokenWithValidArgResolver extends AbstractMessageConverterMethodProcessor implements HandlerMethodArgumentResolver {

    public RequestTokenWithValidArgResolver(List<HttpMessageConverter<?>> converters) {
        super(converters);
    }

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return RequestTokenInfo.class.isAssignableFrom(parameter.getParameterType());
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        HttpServletRequest req = webRequest.getNativeRequest(HttpServletRequest.class);
        if (req == null) {
            return null;
        }
        String addr = req.getRemoteAddr();
        RequestTokenInfo info = new RequestTokenInfo();
        info.setRequestIp(addr);
        String token = req.getHeader("token");
        info.setToken(token);

        String name = Conventions.getVariableNameForParameter(parameter);
        if (binderFactory != null) {
            WebDataBinder binder = binderFactory.createBinder(webRequest, info, name);
            validateIfApplicable(binder, parameter);
            if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
                throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
            }
        }
        return info;
    }
    
    
    /**
     * 不需要支持的方法
     *
     * @param returnType
     * @return
     */
    @Override
    public boolean supportsReturnType(MethodParameter returnType) {
        return false;
    }

    /**
     * 不需要支持的方法
     *
     * @param returnType
     * @return
     */
    @Override
    public void handleReturnValue(Object returnValue, MethodParameter returnType, ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
    }
```

需要自定义解析的实体类

```
public class RequestTokenInfo {

    private String requestIp;

    @NotNull(groups = Auth.class)
    private String token;

    public String getRequestIp() {
        return requestIp;
    }

    public void setRequestIp(String requestIp) {
        this.requestIp = requestIp;
    }

    public String getToken() {
        return token;
    }

    public void setToken(String token) {
        this.token = token;
    }

    public interface Auth{}

}
```

其中Ip是所有接口都拥有的并且都需要的，对于部分不公开的Api需要进行http header参数TOKEN验证。



Controller 方法

```
@RestController
@RequestMapping("/valid")
public class ValidController {

    @PostMapping("/valid")
    public Object valid(@Validated @RequestBody ValidInput input){
        return input;
    }

    @GetMapping("/auth")
    public Object auth(@Validated(RequestTokenInfo.Auth.class) RequestTokenInfo info){
        System.out.println("need auth request"+info);
        return info;
    }


    @GetMapping("/open")
    public Object open(RequestTokenInfo info){
        System.out.println("need auth request"+info);
        return info;
    }
}
```

加入了两个方法：

- /valid/auth：需要进行token验证，注解中配置了group校验组对Token参数进行验证。
- /valid/open：不需要进行token验证。

这样就实现了自定义参数解析集成@Validated 验证。