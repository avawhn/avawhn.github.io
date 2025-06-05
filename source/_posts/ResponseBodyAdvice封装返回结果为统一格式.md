---
title: ResponseBodyAdvice封装返回结果为统一格式
tags:
  - Java
  - SpringBoot
date: 2025-06-04 23:41:55
---


## 作用

与拦截器类的`postHandle`的作用类似，但是`postHandle`并不适合处理`@ResponseBody`和`ResponseEntity`的方法，这些方法是在`HandlerAdapter`中`postHandle`前写入并提交了，这时候在对结果进行处理已经没有用了，而`ResponseBodyAdvice`就是解决这种问题的一个接口，实现这个接口可以对响应结果进行处理。

## 使用

注意：返回类型为`String`时要特殊处理，这时候的`MediaType`是`text/plain`，并不是`application/json`，使用的是`StringHttpMessageConverter`，为了避免不必要的序列化/反序列化过程，这个转换器在写入响应时直接操作`HttpServletResponse`的`Writer`，而不是像其他类型那样使用`MappingJackson2HttpMessageConverter`等

出错的根本原因是先确定的具体`HttpMessageConverter`，然后在调用的`advice`，导致返回类型发生变化，一开始确定的`HttpMessageConverter`无法正常运行，下面是部分源码，位于：`AbstractMessageConverterMethodProcessor.writeWithMessageConverters`， 后续调用了`addDefaultHeaders`，向一个`String`类型的参数传入了`Object`也就是`R`，类型报错，


```java
// AbstractMessageConverterMethodProcessor.writeWithMessageConverters
for (HttpMessageConverter<?> converter : this.messageConverters) {
    GenericHttpMessageConverter genericConverter = (converter instanceof GenericHttpMessageConverter ?
            (GenericHttpMessageConverter<?>) converter : null);
    if (genericConverter != null ?
            ((GenericHttpMessageConverter) converter).canWrite(targetType, valueType, selectedMediaType) :
            converter.canWrite(valueType, selectedMediaType)) {
        body = getAdvice().beforeBodyWrite(body, returnType, selectedMediaType,
                (Class<? extends HttpMessageConverter<?>>) converter.getClass(),
                inputMessage, outputMessage);
        if (body != null) {
            Object theBody = body;
            LogFormatUtils.traceDebug(logger, traceOn ->
                    "Writing [" + LogFormatUtils.formatValue(theBody, !traceOn) + "]");
            addContentDispositionHeader(inputMessage, outputMessage);
            if (genericConverter != null) {
                genericConverter.write(body, targetType, selectedMediaType, outputMessage);
            }
            else {
                ((HttpMessageConverter) converter).write(body, selectedMediaType, outputMessage);
            }
        }
        else {
            if (logger.isDebugEnabled()) {
                logger.debug("Nothing to write: null body");
            }
        }
        return;
    }
}
// AbstractHttpMessageConverter
public final void write(final T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage)
        throws IOException, HttpMessageNotWritableException {

    // ...
    addDefaultHeaders(headers, t, contentType);
    // ...
}
// StringHttpMessageConverter
protected void addDefaultHeaders(HttpHeaders headers, String s, @Nullable MediaType type) throws IOException {
    // ...
}
```

`ResponseBodyAdvice`的使用：

```java
@RestControllerAdvice
public class GlobalResponseHandler implements ResponseBodyAdvice<Object> {
    // 使用 Jackson 的 ObjectMapper 处理 String 类型
    private static final ObjectMapper OBJECT_MAPPER = JsonMapper.builder()
            .findAndAddModules() // 自动注册所有模块
            .build();

    @Override
    public boolean supports(MethodParameter returnType, Class converterType) {
        ResponseWrapper methodAnnotation = returnType.getMethodAnnotation(ResponseWrapper.class);
        if(methodAnnotation != null) {
            return methodAnnotation.value();
        }
        // 判断 Controller 上是否标有 ResponseWrapper 注解
        ResponseWrapper annotation = returnType.getContainingClass().getAnnotation(ResponseWrapper.class);
        if(annotation != null) {
            return annotation.value();
        }
        // 默认启用
        return true;
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType,
                                  Class selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        if (body == null) {
            return R.success();
        }
        if (body instanceof R) {
            return body;
        }
        if (body instanceof String) {
            response.getHeaders().setContentType(MediaType.APPLICATION_JSON);
            try {
                return OBJECT_MAPPER.writeValueAsString(R.success(body));
            } catch (JsonProcessingException e) {
                throw new RuntimeException("请求返回结果JSON序列化失败", e);
            }

        }
        return R.success(body);
    }
}
```


Controller中直接返回原始结果即可，不再需要关注返回类型等操作：

```java
@RestController
@RequestMapping("/user")
public class UserController {
    @GetMapping("/list")
    public List<User> list() {
        /*
        {
            "code": 200,
            "msg": null,
            "data": [
                {
                    "id": 1,
                    "name": "张三",
                    "age": 18
                }
            ]
        }
         */
        List<User> list = new ArrayList<>();
        list.add(new User(1L, "张三", 18));
        return list;
    }


    @PostMapping
    public String add() {
        /*
        {
            "code": 200,
            "msg": null,
            "data": "添加成功"
        }
         */
        return "添加成功";
    }

    @ResponseWrapper(value = false)
    @GetMapping("/noWrap")
    public String noWrap() {
        /*
        noWrap
         */
        return "noWrap";
    }
}
// 这个类的所有未标注ResponseWrapper的方法返回结果不会经过封装
@ResponseWrapper(value = false)
@RestController
@RequestMapping("/nowrap")
public class NoWrapController {
    @RequestMapping("/list")
    public List<User> list() {
        /*
        [
            {
                "id": 1,
                "name": "张三",
                "age": 18
            }
        ]
         */
        List<User> list = new ArrayList<>();
        list.add(new User(1L, "张三", 18));
        return list;
    }
}
```