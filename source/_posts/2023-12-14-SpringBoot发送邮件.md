---
title: SpringBoot发送邮件
date: 2023-12-14 21:47:22
tags: [SpringBoot]
---

- [一、引入依赖](#一引入依赖)
- [二、邮箱服务器](#二邮箱服务器)
- [三、配置](#三配置)
- [四、发送简单消息](#四发送简单消息)
  - [1、使用 SimpleMailMessage](#1使用-simplemailmessage)
  - [2、使用 MimeMessagePreparator](#2使用-mimemessagepreparator)
  - [3、使用 MimeMessageHelper](#3使用-mimemessagehelper)
- [五、发送携带附件与内联资源的邮件](#五发送携带附件与内联资源的邮件)
- [六、使用 FreeMarker 发送 html 格式的邮件](#六使用-freemarker-发送-html-格式的邮件)
  - [1、引入依赖](#1引入依赖)
  - [2、使用 FreeMarker](#2使用-freemarker)
  - [3、创建模板](#3创建模板)
  - [4、发送邮件](#4发送邮件)
- [七、参考资料](#七参考资料)


### 一、引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
</dependency>
```

{% asset_img spring-boot-mail依赖项.png spring-boot-validator包含依赖.png %}

可以看到核心的包只有：jakarta.mail

**org.springframework.mail** 是 Spring 邮件发送的根级包，**MailSender** 是核心接口，**org.springframework.mail.javamail.JavaMailSender** 接口添加了专门的 JavaMail 功能，如对 MailSender 接口（从该接口继承）的 MIME 消息支持。

### 二、邮箱服务器

这里使用网易邮箱

找到如下设置：

{% asset_img 网易邮箱邮件发送申请1.png 网易邮箱邮件发送申请1.png %}

开启 IMAP/SMTP 服务、POP3/SMTP 服务

{% asset_img 网易邮箱邮件发送申请2.png 网易邮箱邮件发送申请2.png %}

扫码发送短信验证后会给出密码，记录下密码，后面需要使用

{% asset_img 网易邮箱邮件发送申请3.png 网易邮箱邮件发送申请3.png %}

服务器地址，后面需要使用

{% asset_img 网易邮箱邮件发送申请4.png 网易邮箱邮件发送申请4.png %}

### 三、配置

```yml
spring:
    mail:
        host: smtp.163.com # 服务器地址，这里使用 SMTP
        username: # 邮箱账号
        password: # 密码
        # 用于属性注入，发送邮件时需要指定 from 与 to
        from: # 你的邮箱
        # 其他配置
        properties:
            "[mail.smtp.connectiontimeout]": 5000
            "[mail.smtp.timeout]": 3000
            "[mail.smtp.writetimeout]": 5000
```

### 四、发送简单消息

#### 1、使用 SimpleMailMessage

```java
@Component
public class EmailUtil {
    @Value("${spring.mail.from}")
    private String from;

    @Resource
    private JavaMailSender javaMailSender;

    public void sendSimpleMessage() {
        SimpleMailMessage message = new SimpleMailMessage();
        message.setFrom(from);
        message.setTo("xx@xx.com");
        message.setSubject("标题");
        message.setText("这是一封使用 SimpleMessage 的简单邮件");
        javaMailSender.send(message);
    }
}
```
{% asset_img 使用SimpleMessage发送简单邮件.png 使用SimpleMessage发送简单邮件.png %}


#### 2、使用 MimeMessagePreparator

```java
public void sendMimeMessagePreparatorMessage() {
    MimeMessagePreparator preparator = new MimeMessagePreparator() {
        @Override
        public void prepare(MimeMessage mimeMessage) throws Exception {
            mimeMessage.setFrom(from);
            mimeMessage.setRecipients(Message.RecipientType.TO, "xx@xx.com");
            mimeMessage.setSubject("MimeMessagePreparator");
            mimeMessage.setText("这是一封使用 MimeMessagePreparator 创建的简单邮件");
        }
    };

    javaMailSender.send(preparator);
}
```

{% asset_img 使用MimeMessagePreparator发送简单邮件.png 使用MimeMessagePreparator发送简单邮件.png %}

#### 3、使用 MimeMessageHelper

```java
public void sendMimeMessageHelperMessage() throws MessagingException {
    MimeMessage msg = javaMailSender.createMimeMessage();
    MimeMessageHelper helper = new MimeMessageHelper(msg);
    helper.setFrom(from);
    helper.setTo("xx@xx.com");
    helper.setSubject("MimeMessageHelper");
    helper.setText("这是一封使用 MimeMessageHelper 创建的简单邮件");
    javaMailSender.send(msg);
}
```
  
{% asset_img 使用MimeMessageHelper发送简单邮件.png 使用MimeMessageHelper发送简单邮件.png %}

### 五、发送携带附件与内联资源的邮件

```java
public void sendAttachmentAndInlineResource() throws MessagingException {
    MimeMessage msg = javaMailSender.createMimeMessage();
    // 使用 true 表示需要 multipart 消息
    MimeMessageHelper helper = new MimeMessageHelper(msg, true);
    helper.setFrom(from);
    helper.setTo("xx@xx.com");
    helper.setSubject("AttachmentAndInlineResource");
    // true 表示内容包含 html 内容
    helper.setText("<html><body><img src='cid:identifier1234' style='width: 100px;'></body></html>", true);
    ResourceLoader resourceLoader = new DefaultResourceLoader();
    // 读取图片
    org.springframework.core.io.Resource resource = resourceLoader.getResource("1.jpg");
    helper.addInline("identifier1234", resource);

    // 读取文档，添加附件
    org.springframework.core.io.Resource resource1 = resourceLoader.getResource("学习资料.doc");
    helper.addAttachment("学习资料", resource1);

    javaMailSender.send(msg);
}
```
   

{% asset_img 发送携带附件与内联资源的邮件.png 发送携带附件与内联资源的邮件.png %}


### 六、使用 FreeMarker 发送 html 格式的邮件

#### 1、引入依赖
```xml
<dependency>
    <groupId>org.freemarker</groupId>
    <artifactId>freemarker</artifactId>
</dependency>
```

#### 2、使用 FreeMarker

```java
public class FreeMarkerTempUtil {
    private static final Configuration CONFIGURATION;

    static {
        CONFIGURATION = new Configuration(Configuration.VERSION_2_3_22);
        CONFIGURATION.setTemplateLoader(
                new SpringTemplateLoader(new DefaultResourceLoader(), "/templates/")
        );
        CONFIGURATION.setDefaultEncoding("UTF-8");
        // 异常处理器
        // CONFIGURATION.setTemplateExceptionHandler(TemplateExceptionHandler.RETHROW);
    }

    public static String renderTemplate(String tempName, Object data) throws IOException, TemplateException {
        Template template = CONFIGURATION.getTemplate(tempName);
        return FreeMarkerTemplateUtils
                .processTemplateIntoString(template, data);
    }
}
```


#### 3、创建模板

在 resources 目录下创建 templates 目录，再创建 test.flth 文件

```html
<html>
<head>
  <title>Welcome!</title>
</head>
<body>
  <h1>Welcome ${user}!</h1>
  <p>Our latest product:
  <a href="${latestProduct.url}">${latestProduct.name}</a>!
</body>
</html>
```

#### 4、发送邮件

```java
public void sendTemplate() throws MessagingException, TemplateException, IOException {
    Map<String, Object> root = new HashMap<>();
    root.put("user", "Big Joe");
    Map<String, Object> latest = new HashMap<>();
    root.put("latestProduct", latest);
    latest.put("url", "products/greenmouse.html");
    latest.put("name", "green mouse");

    // 获取渲染后的页面字符串
    String text = FreeMarkerTempUtil.renderTemplate("test.flth", root);

    MimeMessage message = javaMailSender.createMimeMessage();
    MimeMessageHelper helper = new MimeMessageHelper(message, true);
    helper.setFrom(from);
    helper.setTo("xx@xx.com");
    helper.setSubject("HTML邮件");
    // 必须先添加文本在添加资源
    helper.setText(text, true);
    javaMailSender.send(message);
}
```

{% asset_img 发送html邮件.png 发送html邮件.png %}

### 七、参考资料

[Spring发送邮件](https://docs.spring.io/spring-framework/docs/5.3.31/reference/html/integration.html#mail)
[FreeMarker 中文官方参考手册](http://freemarker.foofun.cn/toc.html)