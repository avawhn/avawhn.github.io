---
title: SpringBoot任务执行和调度
date: 2023-12-19 16:26:09
tags: [SpringBoot]
---

- [一、线程池](#一线程池)
- [二、异步任务](#二异步任务)
  - [1、创建异步方法](#1创建异步方法)
  - [2、异常处理](#2异常处理)
- [二、定时任务](#二定时任务)
  - [1、使用 Spring 创建定时任务](#1使用-spring-创建定时任务)
  - [2、使用 Quartz 创建定时任务](#2使用-quartz-创建定时任务)



### 一、线程池

如果没有自定义 Executor bean，SpringBoot 会自动创建一个 ThreadPoolTaskExecutor，这个线程池会**自动关联到异步任务与 SpringMVC 的请求处理**，可以通过一下参数调整线程池配置：

```yaml
spring:
    task:
        execution:
            pool:
                max-size: 16
                queue-capacity: 100
                keep-alive: "10s"
```

注意：如果自定义了 Executor Bean，那么将会关联到异步任务，但 Spring MVC 支持将不会被配置，因为它需要一个 AsyncTaskExecutor 实现（名为 applicationTaskExecutor）。根据需求，可以将 Executor 更改为 ThreadPoolTaskExecutor，或者同时定义 ThreadPoolTaskExecutor 和封装自定义 Executor 的 AsyncConfigurer。

### 二、异步任务

#### 1、创建异步方法

通过 **@EnableAsync** 开启异步任务，在需要异步执行的方法上加 **@Async** 即可，需要注意方法所属的类要添加到 Spring 的 IOC 容器。

```java
@Component
public class AsyncMethod {
    private static final Logger log = LoggerFactory.getLogger(AsyncMethod.class);

    @Async
    public void say() {
        for (int i = 0; i < 20; i++) {
            log.info("say hello {}", i);
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    }
}
```

获取返回值，可以返回 Future，然后通过 future.get 方法获取。

```java
@Async
public Future<String> future(String prefix) {
    try {
        Thread.sleep(2000);
        return CompletableFuture.completedFuture(prefix + "hello");
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
}
```

#### 2、异常处理

在调用有返回值 Future 的方法，如果有异常，那么调用 future.get 获取返回值的时候就会抛异常，可以处理。但是返回类型是 void 时，异常不会被捕获与传输，可以创建一个 AsyncUncaughtExceptionHandler 来处理异常。

```java
public class MyAsyncUncaughtExceptionHandler implements AsyncUncaughtExceptionHandler {
    private static final Logger log = LoggerFactory.getLogger(MyAsyncUncaughtExceptionHandler.class);

    @Override
    public void handleUncaughtException(Throwable ex, Method method, Object... params) {
        log.error("my exception handle -> ex: {}",ex.toString());
    }
}
```

配置异常处理器

```java
@Configuration
@EnableAsync
public class AsyncConfiguration implements AsyncConfigurer {
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new MyAsyncUncaughtExceptionHandler();
    }
}
```

### 二、定时任务

#### 1、使用 Spring 创建定时任务

可以通过 **@EnableScheduling** 注解将 ThreadPoolTaskScheduler 关联定时任务。

线程池默认使用一个线程，其设置可使用 spring.task.scheduling 命名空间进行微调，如下例所示：

```yaml
spring:
    task:
        scheduling:
            thread-name-prefix: "scheduling-"
            pool:
                size: 2
```

通过 **@Scheduled** 标注在指定的方法上

```java
@Component
public class ScheduledMethod {
    private static final Logger log = LoggerFactory.getLogger(ScheduledMethod.class);

    @Scheduled(fixedRate = 10, timeUnit = TimeUnit.SECONDS)
    public void fixRate() {
        log.info("scheduled method ---> {}", "fixRate");
    }

    @Scheduled(fixedDelay = 2, timeUnit = TimeUnit.SECONDS)
    public void delay2s() {
        log.info("scheduled method ---> {}", "delay2s");
    }
}
```

上面 fixedRate 表示执行的频率，而 fixedDelay 表示执行完延时多长时间再次执行

#### 2、使用 Quartz 创建定时任务

引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

如果 quartz 正常，那么会通过 SchedulerFactoryBean 创建一个 Scheduler。

JobDetail、Calendar、Trigger 会自动检测并关联到 Scheduler。



HelloJob 类

```java
public class HelloJob implements Job {
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        System.out.println("hello job");
    }
}
```

配置 Job 与 Trigger，Spring 会自动检测并关联 Scheduler 并执行，不需要手动启动。

```java
@Configuration
public class QuartzConfig {
    @Bean(name = "helloJob")
    public JobDetail helloJob() {
        return JobBuilder.newJob(HelloJob.class)
                .withIdentity("helloJob")
                .storeDurably()
                .build();
    }

    @Bean(name = "helloTrigger")
    public Trigger helloTrigger() {
        return TriggerBuilder.newTrigger()
                .forJob(helloJob())
                .withIdentity("helloTrigger")
                .withSchedule(
                        SimpleScheduleBuilder.simpleSchedule()
                                .withIntervalInSeconds(10)
                                .repeatForever()
                )
                .build();
    }
}
```

除了上面的方法，也可以正常创建 Job、Trigger 然后通过 scheduler.scheduleJob(jobDetail, trigger) 创建定时任务。

默认 JobStore 基于内存，也可以配置为基于 JDBC

```yaml
spring:
  quartz:
    job-store-type: "jdbc"
```

使用 JDBC 存储时，可在启动时初始化模式，如下例所示：

```yaml
spring:
  quartz:
    jdbc:
      initialize-schema: "always"
```

>默认情况下，数据库通过使用 Quartz 库提供的标准脚本进行检测和初始化。这些脚本会删除现有表格，并在每次重启时删除所有触发器。也可以通过设置 spring.quartz.jdbc.schema 属性来提供自定义脚本。


要让 Quartz 使用应用程序主数据源以外的数据源，可声明一个数据源 Bean，并用 @QuartzDataSource 对其 @Bean 方法进行注解。这样做可以确保 SchedulerFactoryBean 和模式初始化都使用特定于 Quartz 的数据源。同样，若要让 Quartz 使用应用程序主事务管理器（TransactionManager）之外的事务管理器（TransactionManager），可声明一个事务管理器 Bean，并用 @QuartzTransactionManager 对其 @Bean 方法进行注解。

默认情况下，通过配置创建的作业不会覆盖已从持久作业存储中读取的已注册作业。要启用覆盖现有作业定义，请设置 spring.quartz.overwrite-existing-jobs 属性。

Quartz Scheduler 配置可使用 spring.quartz 属性和 SchedulerFactoryBeanCustomizer Bean 进行自定义，后者允许对 SchedulerFactoryBean 进行编程式自定义。高级 Quartz 配置属性可使用 spring.quartz.properties.*.Quartz 属性来定制。

>特别是，由于 Quartz 提供了一种通过 spring.quartz.properties 配置调度器的方法，因此 Executor Bean 与调度器无关。如果您需要自定义任务执行器，请考虑实现 SchedulerFactoryBeanCustomizer。

Bean 注入可以通过 setter 注入

```java
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;

import org.springframework.scheduling.quartz.QuartzJobBean;

public class MySampleJob extends QuartzJobBean {

    private MyService myService;

    private String name;

    // setter 注入 myService
    public void setMyService(MyService myService) {
        this.myService = myService;
    }

    // setter 注入 name
    public void setName(String name) {
        this.name = name;
    }

    @Override
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        this.myService.someMethod(context.getFireTime(), this.name);
    }

}
```