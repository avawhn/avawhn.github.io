---
title: SpringBoot参数校验
date: 2023-12-12 20:57:22
tags: [SpringBoot]
---

- [一、引入依赖](#一引入依赖)
- [二、URL 校验](#二url-校验)
- [三、实体类校验](#三实体类校验)
  - [1、测试](#1测试)
  - [2、valid 与 validated 的区别](#2valid-与-validated-的区别)
- [四、组校验](#四组校验)
- [五、自定义校验规则](#五自定义校验规则)
  - [1、创建约束注解](#1创建约束注解)
  - [2、创建约束校验器](#2创建约束校验器)
  - [3、应用](#3应用)
- [六、配合全局异常使用](#六配合全局异常使用)
- [七、手动校验](#七手动校验)
  - [1、校验整个实体类](#1校验整个实体类)
  - [2、校验实体类中的某个属性](#2校验实体类中的某个属性)
- [八、将错误信息作为入参接收（验证失败不抛异常）](#八将错误信息作为入参接收验证失败不抛异常)
- [九、自定义validator](#九自定义validator)

### 一、引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

{% asset_img spring-boot-validator包含依赖.png spring-boot-validator包含依赖.png %}

可以看到里面包括

-   jakarta.validation-api：JSR 303 规范
-   [hibernate-validator](https://docs.jboss.org/hibernate/validator/4.2/reference/zh-CN/html_single/)：参数校验的具体实现

### 二、URL 校验

```java
@RestController
@Validated
public class TestController {
    @RequestMapping("/url")
    public void url(@Min(5) @RequestParam("id") Long id) {
        System.out.println(id);
    }
    @RequestMapping("/{id}")
    public void pathVariable(@Min(5) @PathVariable("id") Long id) {
        System.out.println(id);
    }
}
```

测试

-   Url：http://127.0.0.1:8080/url?id=1
-   PathVariable：http://127.0.0.1:8080/1

{% asset_img url与pathvariable参数校验结果.png url与pathvariable参数校验结果.png %}

注意：@Validated 只有标注在类上才能对 URL 或者 PathVariable 的参数生效

### 三、实体类校验

#### 1、测试

实体类

```java
public class User {
    @NotNull(message = "id不能为空")
    private Long id;

    @NotBlank(message = "name不能为空")
    private String name;

    @Min(value = 0, message = "年龄最小为0")
    private Integer age;

    @Email(message = "不符合邮箱格式")
    private String email;

    @Length(min = 8, max = 18, message = "密码长度必须在8-18位之间")
    private String password;

    // constructor、setter and getter
}
```

校验

```java
@RestController
@Validated
public class TestController {
    @RequestMapping("/class")
    public void clazz(@Validated @RequestBody User user) {
        System.out.println(user);
    }
}
```

当校验实体类时，方法参数中的 @Validated 也可换为 @Valid

测试参数

```json
{
    "id": null,
    "name": "",
    "age": -1,
    "email": "xx",
    "password": "123"
}
```

测试结果：

{% codeblock 控制台日志 lang:log %}
WARN 16992 --- [nio-8080-exec-1] .w.s.m.s.DefaultHandlerExceptionResolver :
Resolved [org.springframework.web.bind.MethodArgumentNotValidException: Validation failed for argument [0] in public void com.whn.validate.demo.controller.TestController.clazz(com.whn.validate.demo.domain.User) with 5 errors:
[

    Field error in object 'user' on field 'name': rejected value []; codes [NotBlank.user.name,NotBlank.name,NotBlank.java.lang.String,NotBlank];
    arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [user.name,name]; arguments []; default message [name]];
    default message [name 不能为空]

]
[

    Field error in object 'user' on field 'email': rejected value [xx]; codes [Email.user.email,Email.email,Email.java.lang.String,Email];

    arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [user.email,email]; arguments []; default message [email],[Ljavax.validation.constraints.Pattern$Flag;@5c928100,.*];
    default message [不符合邮箱格式]]

    [Field error in object 'user' on field 'password': rejected value [123]; codes [Length.user.password,Length.password,Length.java.lang.String,Length];
    arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [user.password,password]; arguments []; default message [password],18,8];
    default message [密码长度必须在8-18位之间]]

    [Field error in object 'user' on field 'id': rejected value [null]; codes [NotNull.user.id,NotNull.id,NotNull.java.lang.Long,NotNull];
    arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [user.id,id]; arguments []; default message [id]];
    default message [id不能为空]]

    [Field error in object 'user' on field 'age': rejected value [-1]; codes [Min.user.age,Min.age,Min.java.lang.Integer,Min];
    arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [user.age,age]; arguments []; default message [age],0];
    default message [年龄最小为0]]

]
{% endcodeblock %}

{% asset_img 注解校验实体类页面测试结果.png 注解校验实体类页面测试结果.png %}

可以看到抛出了 MethodArgumentNotValidException 异常，并且参数全部进行了校验

其他内置的约束条件，可以查看 [Hibernate Validator 官网](https://docs.jboss.org/hibernate/validator/4.2/reference/zh-CN/html_single/#validator-defineconstraints-builtin)

#### 2、valid 与 validated 的区别

- valid 属于 JSR 303 规范，而 validated 属于 spring
- valid 不支持组校验，validated 支持
- valid 可以用于方法、字段、构造函数、方法参数、类型，validated 可以用于方法、参数、类型



**注意事项**
- 只有 validated 标注在 controller 类上才会对 @PathVariable 或者 @RequestParam 起作用，valid 标注在类上也不会对这两种参数起作用
- 当 validated 标注在类上时，使用 valid 对实体类进行校验，成功时会校验两次，失败直接返回了，使用 validated 校验一次
- 当 validated 标注在类上时，不会对实体类进行校验，必须再使用 validated 或者 valid 作为入口


### 四、组校验

需要校验的类

```java
public class Person {
    @Min(value = 5, message = "g1组要求最小为5", groups = {G1Check.class})
    @Min(value = 10, message = "g2组要求最小为10", groups = {G2Check.class})
    private Integer age;
    // constructor、setter and getter
}
```

可以看到 G1Check 要求最小值为 5，G2Check 要求最小值为 10

测试接口

```java
@RestController
@Validated
public class TestController {
    @RequestMapping("/g1")
    public void g1(@Validated(G1Check.class) @RequestBody Person person) {
        System.out.println("g1");
        System.out.println(person);
    }
    @RequestMapping("/g2")
    public void g2(@Validated(G2Check.class) @RequestBody Person person) {
        System.out.println("g2");
        System.out.println(person);
    }
}
```

使用相同的参数校验

```json
{
    "age": 8
}
```

预期结果是 g1 通过，g2 不通过

g1 测试结果：

{% asset_img 组校验g1测试结果.png  组校验g1测试结果.png %}

g2 测试结果

{% asset_img 组校验g2测试结果.png  组校验g2测试结果.png %}

```log
WARN 7372 --- [nio-8080-exec-4] .w.s.m.s.DefaultHandlerExceptionResolver :
Resolved [org.springframework.web.bind.MethodArgumentNotValidException:
Validation failed for argument [0] in public void com.whn.validate.demo.controller.TestController.g2(com.whn.validate.demo.domain.Person):
[
    Field error in object 'person' on field 'age':
        rejected value [8];
        codes [Min.person.age,Min.age,Min.java.lang.Integer,Min];
    arguments [org.springframework.context.support.DefaultMessageSourceResolvable:
        codes [person.age,age];
        arguments [];
        default message [age],10
    ];
    default message [g2组要求最小为10]]
]
```

可以看到 g1 参数校验通过，g2 校验失败，抛出 MethodArgumentNotValidException 异常，符合预期。

有关组校验序列以及重定义默认校验组，可以查看[官网](https://docs.jboss.org/hibernate/validator/4.2/reference/zh-CN/html_single/#d0e965)

### 五、自定义校验规则

流程有 3 个步骤：

1. 创建约束注解
2. 创建约束校验器
3. 应用约束条件

#### 1、创建约束注解

枚举

```java
public enum GenderEnum {
    MALE, FEMALE;
}
```

约束注解

```java
@Target({METHOD, FIELD, ANNOTATION_TYPE})
@Retention(RUNTIME)
@Constraint(validatedBy = GenderCheckCaseValidator.class)
@Documented
public @interface GenderCheckCase {
    String message() default "{性别错误}";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

    GenderEnum value();
}
```

-   validatedBy 指向自定义的校验器
-   payload，可以通过此属性来给约束条件指定严重级别. 这个属性并不被 API 自身所使用

#### 2、创建约束校验器

```java
public class GenderCheckCaseValidator implements ConstraintValidator<GenderCheckCase, GenderEnum> {

    private GenderCheckCase constraintAnnotation;

    @Override
    public void initialize(GenderCheckCase constraintAnnotation) {
        this.constraintAnnotation = constraintAnnotation;
    }

    @Override
    public boolean isValid(GenderEnum value, ConstraintValidatorContext context) {
        if (value == null) {
            return true;
        }
        if (constraintAnnotation.value() != value) {
            // 不禁用默认模板，会多一条消息
            context.disableDefaultConstraintViolation();
            context
                    .buildConstraintViolationWithTemplate("性别必须是 {value}")
                    .addConstraintViolation();
        }
        return constraintAnnotation.value() == value;
    }
}
```

对于 value 为空的情况下应该返回 true，因为这种情况应该交由 @NotNull 控制

上面采用了自定义错误信息模板，必须调用 addConstraintViolation，这个新的 constraint violation 才会被真正的创建，错误信息解析可以查看[官网](https://docs.jboss.org/hibernate/validator/4.2/reference/zh-CN/html_single/#section-message-interpolation)

#### 3、应用

```java
public class Person {
    @Min(value = 5, message = "g1组要求最小为5", groups = {G1Check.class})
    @Min(value = 10, message = "g2组要求最小为10", groups = {G2Check.class})
    private Integer age;

    @GenderCheckCase(GenderEnum.MALE)
    private GenderEnum genderEnum;

    // constructor、setter and getter
}
```

测试接口

```java
@RestController
@Validated
public class TestController {
    @RequestMapping("/custom")
    public void custom(@Validated @RequestBody Person person) {
        System.out.println("success");
    }
}
```

测试参数：

```json
{
    "genderEnum": "  FEMALE"
}
```

测试结果：

```log
WARN 19408 --- [nio-8080-exec-3] .w.s.m.s.DefaultHandlerExceptionResolver :
Resolved
[
    org.springframework.web.bind.MethodArgumentNotValidException:

    Validation failed for argument [0] in public void com.whn.validate.demo.controller.TestController.custom(com.whn.validate.demo.domain.Person):
    [
        Field error in object 'person' on field 'genderEnum': rejected value [FEMALE];

        codes [GenderCheckCase.person.genderEnum,GenderCheckCase.genderEnum,GenderCheckCase.com.whn.validate.demo.validate.GenderEnum,GenderCheckCase];

        arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [person.genderEnum,genderEnum]; arguments [];

        default message [genderEnum],MALE];

        default message [性别必须是 MALE]
    ]
]
```

### 六、配合全局异常使用

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public String methodArgumentNotValidExceptionHandle(MethodArgumentNotValidException e) {
        StringBuffer sb = new StringBuffer();
        e.getFieldErrors().forEach(fe -> {
            sb.append(fe.getField()).append("内容错误：").append(fe.getDefaultMessage()).append("\n");
        });
        return sb.toString();
    }
}
```
测试之前的实体类校验，参数：

```json
{
    "id": null,
    "name":"",
    "age":-1,
    "email":"xx",
    "password":"123"
}
```

返回结果

```text
password内容错误：密码长度必须在8-18位之间
email内容错误：不符合邮箱格式
name内容错误：name不能为空
age内容错误：年龄最小为0
id内容错误：id不能为空
```

### 七、手动校验

#### 1、校验整个实体类

```java
@RestController
@RequestMapping("/handle")
public class HandleController {
    private static final Validator VALIDATOR = Validation.buildDefaultValidatorFactory().getValidator();

    @RequestMapping("/t1")
    public void validate(@RequestBody User user) {
        System.out.println(user);
        Set<ConstraintViolation<User>> constraintViolations = VALIDATOR.validate(user);
        System.out.println(constraintViolations.size());
        for (ConstraintViolation<User> constraintViolation : constraintViolations) {
            System.out.println(constraintViolation.getMessage());
        }
    }
}

```

测试参数

```json
{
    "id": null,
    "name": "",
    "age": -1,
    "email": "xx",
    "password": "123"
}
```

测试结果

{% asset_img 手动校验测试结果.png 手动校验测试结果.png %}

#### 2、校验实体类中的某个属性

```java
@RequestMapping("/one")
public void validateOneAttr(@RequestBody User user) {
    System.out.println(user);
    Set<ConstraintViolation<User>> constraint = VALIDATOR.validateProperty(user, "id");
    for (ConstraintViolation<User> c : constraint) {
        System.out.println(c.getMessage());
    }
}
```

测试参数

```json
{
    "id": null,
    "name": "",
    "age": -1,
    "email": "xx",
    "password": "123"
}
```

测试结果

{% asset_img 手动校验某个参数测试结果.png 手动校验某个参数测试结果.png %}

通过 validateProperty()可以对一个给定实体对象的单个属性进行校验. 其中属性名称需要符合 JavaBean 规范中定义的属性名称.

例如, Validator.validateProperty 可以被用在把 Bean Validation 集成进 JSF 2 中的时候使用

### 八、将错误信息作为入参接收（验证失败不抛异常）

[参数官网](https://docs.spring.io/spring-framework/docs/5.2.25.RELEASE/spring-framework-reference/web.html#mvc-ann-arguments)

|Controller method argument|Description|
|---|--|
| Errors, BindingResult | For access to errors from validation and data binding for a command object (that is, a @ModelAttribute argument) or errors from the validation of a @RequestBody or @RequestPart arguments. You must declare an Errors, or BindingResult argument immediately after the validated method argument. |

```java
@RequestMapping("/binder")
public void binder(@Valid @RequestBody Person person, BindingResult result) {
    StringBuffer sb = new StringBuffer();
    List<FieldError> errors = result.getFieldErrors();
    errors.forEach(e -> {
        sb.append(e.getField()).append("内容错误").append(", 错误信息：").append(e.getDefaultMessage());
    });
    System.out.println(sb.toString());
}
```
   
### 九、自定义validator

自定义 validator

```java
public class PersonValidator implements Validator {
    @Override
    public boolean supports(Class<?> clazz) {
        return Person.class.equals(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        if (errors.getErrorCount() == 0) {
            return;
        }
        StringBuffer sb = new StringBuffer();
        errors.getFieldErrors().forEach(e -> {
            sb.append(e.getField()).append(" 数据错误").append(", 错误信息：").append(e.getDefaultMessage());
        });
        throw new PensonParamErrorException(sb.toString());
    }
}
```

异常类

```java
public class PensonParamErrorException extends RuntimeException{
    public PensonParamErrorException(String message) {
        super(message);
    }
}
```

```java
@RestController
public class TestController {
    @InitBinder
    protected void initBinder(WebDataBinder binder) {
        binder.addValidators(new PersonValidator());
    }
    @RequestMapping("/validator")
    public void validator(@Validated @RequestBody Person person) {
    }
}
```

测试结果：

{% asset_img 自定义validator测试结果.png 自定义validator测试结果.png %}

