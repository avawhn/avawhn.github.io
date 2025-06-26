---
title: SpringBoot启动流程
tags: [Java, SpringBoot]
---

- [1. SpringBoot的Main方法](#1-springboot的main方法)
- [2. SpringApplication.run](#2-springapplicationrun)
  - [2.1. 实例化SpringApplication](#21-实例化springapplication)
  - [2.2. run](#22-run)
    - [2.2.1. prepareEnvironment准备环境](#221-prepareenvironment准备环境)
    - [2.2.2. createApplicationContext实例化应用上下文](#222-createapplicationcontext实例化应用上下文)
    - [2.2.3. prepareContext准备上下文](#223-preparecontext准备上下文)
    - [2.2.4. refreshContext刷新上下文](#224-refreshcontext刷新上下文)
    - [2.2.5. afterRefresh刷新的后置处理](#225-afterrefresh刷新的后置处理)
- [3. @SpringBootApplication](#3-springbootapplication)
  - [3.1. @SpringBootConfiguration](#31-springbootconfiguration)
  - [3.2. @EnableAutoConfiguration](#32-enableautoconfiguration)
    - [3.2.1. @AutoConfigurationPackage](#321-autoconfigurationpackage)
    - [3.2.2. AutoConfigurationImportSelector](#322-autoconfigurationimportselector)
  - [3.3. @ComponentScan](#33-componentscan)

## 1. SpringBoot的Main方法

SpringBoot的启动可以分为`@SpringBootApplication`注解以及`SpringApplication.run`，分别查看这两部分的作用，这里以`2.7.5`版本进行查看。

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

## 2. SpringApplication.run

{% asset_img SpringApplication的run方法流程.png %}

### 2.1. 实例化SpringApplication

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 根据某些类是否存在进行判定
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 获取 META-INF/spring.factories 文件中以 BootstrapRegistryInitializer 全类名为键的类的实例
    this.bootstrapRegistryInitializers = new ArrayList<>(
            getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
    // 获取 META-INF/spring.factories 文件中以 ApplicationContextInitializer 全类名为键的类的实例
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 获取 META-INF/spring.factories 文件中以 ApplicationListener 全类名为键的类的实例
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 获取 main 方法所属类
    this.mainApplicationClass = deduceMainApplicationClass();
}
// 获取 META-INF/spring.factories 文件中以 type 类名为键的类的实例
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader();
    // 获取 META-INF/spring.factories 文件中以 type 类名为键的值（全类名）
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 实例化
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    // 排序
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

下面是`spring-boot`的`spring.factories`文件关于`ApplicationContextInitializer`的配置：

```property
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
org.springframework.boot.rsocket.context.RSocketPortInfoApplicationContextInitializer,\
org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
```

### 2.2. run

```java
public ConfigurableApplicationContext run(String... args) {
    // 获取当前时间，用于计时
    long startTime = System.nanoTime();
    // 创建引导上下文
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();
    ConfigurableApplicationContext context = null;
    // 配置java.awt.headless=true，无头模式，无键盘、鼠标以及显示，也就是无图形化模式
    configureHeadlessProperty();
    // 获取应用运行监听器，在运行周期指定节点发布对应事件，例如starting(启动中)、started（已运行)
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 发布应用开始启动事件
    listeners.starting(bootstrapContext, this.mainApplicationClass);
    try {
        // 解析并封装命令行参数，例如 java -jar demo.jar --server.port=9090 中的 --server.port=9090
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 准备环境，包括Serverlet配置初始参数、Serverlet上下文初始参数、JVM参数、系统环境变量、命令行参数、profile、配置等参数
        // 绑定环境到应用
        ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
        // 配置忽略BeanInfo
        configureIgnoreBeanInfo(environment);
        // 打印banner
        Banner printedBanner = printBanner(environment);
        // 创建应用上下文，AnnotationConfigServletWebServerApplicationContext
        context = createApplicationContext();
        context.setApplicationStartup(this.applicationStartup);
        // 准备上下文
        prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
        // 刷新上下文
        refreshContext(context);
        // 刷新后置处理
        afterRefresh(context, applicationArguments);
        Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), timeTakenToStartup);
        }
        // 发布应用已启动事件
        listeners.started(context, timeTakenToStartup);
        // 代用Runner的run方法
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, listeners);
        throw new IllegalStateException(ex);
    }
    try {
        Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
        // 发布应用就绪事件
        listeners.ready(context, timeTakenToReady);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```

#### 2.2.1. prepareEnvironment准备环境

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
        DefaultBootstrapContext bootstrapContext, ApplicationArguments applicationArguments) {
    // 创建Environment对象并获取Serverlet配置初始参数、Serverlet上下文初始参数、JVM参数、系统环境变量
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    // 配置PropertySources和Profiles信息
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    // 将每个PropertySource转换(适配)为一个ConfigurationPropertySource，并允许经典的PropertySourcesPropertyResolver调用使用配置属性名进行解析。
    // 将ConfigurationPropertySource添加到PropertySource列表第一位
    ConfigurationPropertySources.attach(environment);
    // 发布环境就绪事件, application.yml数据的获取就是在事件发布后监听到该事件并进行获取
    listeners.environmentPrepared(bootstrapContext, environment);
    DefaultPropertiesPropertySource.moveToEnd(environment);
    Assert.state(!environment.containsProperty("spring.main.environment-prefix"),
            "Environment prefix cannot be set via properties.");
    // 将环境变量绑定到应用
    bindToSpringApplication(environment);
    if (!this.isCustomEnvironment) {
        // 转换为ApplicationServletEnvironment
        EnvironmentConverter environmentConverter = new EnvironmentConverter(getClassLoader());
        environment = environmentConverter.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```

#### 2.2.2. createApplicationContext实例化应用上下文

```java
public AnnotationConfigServletWebServerApplicationContext() {
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

- AnnotatedBeanDefinitionReader: 编程式注册Bean
- ClassPathBeanDefinitionScanner: 类路径扫描Bean

#### 2.2.3. prepareContext准备上下文

```java
private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context,
        ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments, Banner printedBanner) {
    // 为应用上下文设置环境
    context.setEnvironment(environment);
    // 后置处理应用上下文
    postProcessApplicationContext(context);
    // 对每个ApplicationContextInitializer初始化上下文
    applyInitializers(context);
    // 发布上下文就绪事件
    listeners.contextPrepared(context);
    // 发布引导上下文关闭事件
    bootstrapContext.close(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    // Add boot specific singleton beans
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    // 注册参数对象为单例Bean
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    if (beanFactory instanceof AbstractAutowireCapableBeanFactory) {
        // 设置是否允许循环依赖(引用)
        ((AbstractAutowireCapableBeanFactory) beanFactory).setAllowCircularReferences(this.allowCircularReferences);
        // 设置BeanDefinition是否允许覆盖
        if (beanFactory instanceof DefaultListableBeanFactory) {
            ((DefaultListableBeanFactory) beanFactory)
                    .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
        }
    }
    // 延迟初始化
    if (this.lazyInitialization) {
        // 延迟初始化Bean工厂后置处理器
        context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
    }
    // 添加后置处理器
    context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context));
    // Load the sources
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    // 加载Bean到应用上下文
    load(context, sources.toArray(new Object[0]));
    // 发布上下文已加载事件
    listeners.contextLoaded(context);
}
```

#### 2.2.4. refreshContext刷新上下文

这部分属于spring的核心，后续再进行补充

#### 2.2.5. afterRefresh刷新的后置处理

后续补充

## 3. @SpringBootApplication

{% asset_img SpringBootApplication注解.png %}

去除掉元注解后内容如下：

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    // ...
}
```

可以看到由`SpringBootConfiguration`、`EnableAutoConfiguration`、`ComponentScan`组成。

### 3.1. @SpringBootConfiguration

```java
@Configuration
@Indexed
public @interface SpringBootConfiguration {
    // ...
}
```

`SpringBootConfiguration`组成如下：

- Configuration：标识这是一个配置类
- [Indexed](https://docs.spring.io/spring-framework/docs/5.2.25.RELEASE/javadoc-api/)：用于构建扫描索引，提升应用启动时的类扫描速度

### 3.2. @EnableAutoConfiguration

```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    // ...
}
```

`@EnableAutoConfiguration`注解上用`@AutoConfigurationPackage`进行标注，并通过`@Import`导入了`AutoConfigurationImportSelector`类

#### 3.2.1. @AutoConfigurationPackage

```java
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
    // ...
}
```

`@Import`注解相当于xml配置文件中的`<import/>`元素，允许导入`@Configuration`配置类、`ImportSelector`和`ImportBeanDefinitionRegistrar`实现类以及常规组件类。

可以看到这里导入了`AutoConfigurationPackages.Registrar`类，类层次结构如图所示：

{% asset_img Registrar类层次结构.png %}

```java
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        register(registry, new PackageImports(metadata).getPackageNames().toArray(new String[0]));
    }
    @Override
    public Set<Object> determineImports(AnnotationMetadata metadata) {
        return Collections.singleton(new PackageImports(metadata));
    }
}

/**
 * 以编程方式注册自动配置包名称。后续调用将把给定的包名添加到已注册的包名中。您可以使用此方法手动定义将用于给定 BeanDefinitionRegistry 的基础包。一般情况下，建议您不要直接调用此方法，而是采用默认约定，即通过 @EnableAutoConfiguration 配置类或类来设置包名。
 */
public static void register(BeanDefinitionRegistry registry, String... packageNames) {
    // BEAN: org.springframework.boot.autoconfigure.AutoConfigurationPackages
    if (registry.containsBeanDefinition(BEAN)) {
        BasePackagesBeanDefinition beanDefinition = (BasePackagesBeanDefinition) registry.getBeanDefinition(BEAN);
        beanDefinition.addBasePackages(packageNames);
    }
    else {
        registry.registerBeanDefinition(BEAN, new BasePackagesBeanDefinition(packageNames));
    }
}
```

其实就是注册包名，用于后续扫描使用，例如JPA扫描entity。

#### 3.2.2. AutoConfigurationImportSelector



`AutoConfigurationImportSelector`类层次结构如图所示：

{% asset_img AutoConfigurationImportSelector类层次结构.png %}

可以看到这个类实现了`DeferredImportSelector`类型，这个接口的主要方法如下：

```java
public interface DeferredImportSelector extends ImportSelector {
    // ...
    interface Group {
        /**
         * Process the {@link AnnotationMetadata} of the importing @{@link Configuration}
         * class using the specified {@link DeferredImportSelector}.
         */
        void process(AnnotationMetadata metadata, DeferredImportSelector selector);

        class Entry {
            // ...
        }
    }
}
```

`Group.process`是主要运行的方法是，而不是`selectImports`方法，这与之前的版本不同，具体从是哪个版本进行的改动不进行细究。

在`org.springframework.context.annotation.ConfigurationClassParser.processImports`中可以发现对`DeferredImportSelector`类型进行了特殊处理：

```java
if (selector instanceof DeferredImportSelector) {
    this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
}
else {
    String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
    Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames, exclusionFilter);
    processImports(configClass, currentSourceClass, importSourceClasses, exclusionFilter, false);
}

private class DeferredImportSelectorHandler {
    @Nullable
    private List<DeferredImportSelectorHolder> deferredImportSelectors = new ArrayList<>();

    public void handle(ConfigurationClass configClass, DeferredImportSelector importSelector) {
        DeferredImportSelectorHolder holder = new DeferredImportSelectorHolder(configClass, importSelector);
        if (this.deferredImportSelectors == null) {
            // 这里与process中处理一致, 注册->处理组导入
            DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
            handler.register(holder);
            handler.processGroupImports();
        }
        else {
            this.deferredImportSelectors.add(holder);
        }
    }
    // 实际的处理方法
    public void process() {
        List<DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
        this.deferredImportSelectors = null;
        try {
            if (deferredImports != null) {
                DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
                deferredImports.sort(DEFERRED_IMPORT_COMPARATOR);
                // 循环注册Import
                deferredImports.forEach(handler::register);
                // 处理组导入
                handler.processGroupImports();
            }
        }
        finally {
            this.deferredImportSelectors = new ArrayList<>();
        }
    }
}

public void processGroupImports() {
    for (DeferredImportSelectorGrouping grouping : this.groupings.values()) {
        Predicate<String> exclusionFilter = grouping.getCandidateFilter();
        // 这里调用的是Group.getImports
        grouping.getImports().forEach(entry -> {
            // ...
        });
    }
}
public Iterable<Group.Entry> getImports() {
    for (DeferredImportSelectorHolder deferredImport : this.deferredImports) {
        this.group.process(deferredImport.getConfigurationClass().getMetadata(),
                deferredImport.getImportSelector());
    }
    return this.group.selectImports();
}
```

再回到`AutoConfigurationImportSelector`，查看`Group.process`方法：

```java
// 处理（过滤、排序等）并返回 autoConfigurationEntries
public Iterable<Entry> selectImports() {
    // ...
}

public void process(AnnotationMetadata annotationMetadata, DeferredImportSelector deferredImportSelector) {
    // 校验
    Assert.state(deferredImportSelector instanceof AutoConfigurationImportSelector,
            () -> String.format("Only %s implementations are supported, got %s",
                    AutoConfigurationImportSelector.class.getSimpleName(),
                    deferredImportSelector.getClass().getName()));
    // 根据元数据构建Entry
    AutoConfigurationEntry autoConfigurationEntry = ((AutoConfigurationImportSelector) deferredImportSelector)
            .getAutoConfigurationEntry(annotationMetadata);
    this.autoConfigurationEntries.add(autoConfigurationEntry);
    for (String importClassName : autoConfigurationEntry.getConfigurations()) {
        this.entries.putIfAbsent(importClassName, annotationMetadata);
    }
}
// 获取自动配置的Entry
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }
    // 获取EnableAutoConfiguration注释的属性
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    // 获取候选(自动)配置类名
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    // 移除重复项
    configurations = removeDuplicates(configurations);
    // 移除排除项
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    checkExcludedClasses(configurations, exclusions);
    configurations.removeAll(exclusions);
    // 过滤
    configurations = getConfigurationClassFilter().filter(configurations);
    // 发布自动配置类导入事件
    fireAutoConfigurationImportEvents(configurations, exclusions);
    // 封装并返回
    return new AutoConfigurationEntry(configurations, exclusions);
}

// 获取候选配置类名
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    // getSpringFactoriesLoaderFactoryClass这里是EnableAutoConfiguration.class
    // SpringFactoriesLoader.loadFactoryNames获取META-INF/spring.factories中以Key为键的值, 这里EnableAutoConfiguration.class没有, 所以configurations是一个空列表
    List<String> configurations = new ArrayList<>(
            SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader()));
    // 加载org.springframework.boot.autoconfigure.AutoConfiguration.imports中的自动配置类
    ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader()).forEach(configurations::add);
    Assert.notEmpty(configurations,
            "No auto configuration classes found in META-INF/spring.factories nor in META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports. If you "
                    + "are using a custom packaging, make sure that file is correct.");
    return configurations;
}
```

可以发现这与网上的有所不同，因为SpringBoot从2.7版本对自动配置类注册进行了改动，这里截取[官方文档](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.7-Release-Notes#auto-configuration-registration)的部分内容：

{% asset_img SpringBoot2.7自动配置注册说明.png %}

这里起主要作用的方法是`ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader())`，这也是新版本各个starter的实现方法, 通过`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`文件中指定自动配置类，进行配置，例如`mybatis-spring-boot-starter`，依赖中有一个`mybatis-spring-boot-autoconfigure`，这个JAR包下就定义了该文件，内容如下：

```text
org.mybatis.spring.boot.autoconfigure.MybatisLanguageDriverAutoConfiguration
org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration
```

### 3.3. @ComponentScan

设置组件扫描路径，作用于xml配置文件中的`<context:component-scan>`相同。如果不指定`basePackages`，默认从标注的类开始，也就是当前包以及子包，这也是为什么将启动类放在最外层。