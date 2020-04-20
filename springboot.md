# Springboot运行流程

## 构造方法

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
  // 资源加载器
  this.resourceLoader = resourceLoader;
  Assert.notNull(primarySources, "PrimarySources must not be null");
  // primarySources 主启动类的 class
  this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
  // 判断web容器的类型
  this.webApplicationType = WebApplicationType.deduceFromClasspath();
  // 扫描当前路径下META-INF/spring.factories文件 加载ApplicationContextInitializer类型的初始化器
  setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
  // 扫描当前路径下META-INF/spring.factories文件 加载ApplicationListener 监听器
  setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
  // 设置主启动类
  this.mainApplicationClass = deduceMainApplicationClass();
}
```

### deduceFromClasspath

```java
static WebApplicationType deduceFromClasspath() {
  if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
      && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
    //  项目中存在 org.springframework.web.reactive.DispatcherHandler
    //  不存在 org.springframework.web.servlet.DispatcherServlet和org.glassfish.jersey.servlet.ServletContainer
    // reactive 容器
    return WebApplicationType.REACTIVE;
  }
  for (String className : SERVLET_INDICATOR_CLASSES) {
     //项目中不存在javax.servlet.Servlet和org.springframework.web.context.ConfigurableWebApplicationContext
    if (!ClassUtils.isPresent(className, null)) {
      return WebApplicationType.NONE;
    }
  }
  // servlet容器
  return WebApplicationType.SERVLET;
}
```

### deduceMainApplicationClass

```java
private Class<?> deduceMainApplicationClass() {
  try {
    StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
    for (StackTraceElement stackTraceElement : stackTrace) {
      // 从栈中查找main方法
      if ("main".equals(stackTraceElement.getMethodName())) {
        return Class.forName(stackTraceElement.getClassName());
      }
    }
  }
  catch (ClassNotFoundException ex) {
    // Swallow and continue
  }
  return null;
}
```

## run(args)

```java
public ConfigurableApplicationContext run(String... args) {
  StopWatch stopWatch = new StopWatch();
  stopWatch.start();
  ConfigurableApplicationContext context = null;
  Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
  configureHeadlessProperty();
  // 获取事件监听器并启动执行
  SpringApplicationRunListeners listeners = getRunListeners(args);
  listeners.starting();
  try {
    // 用DefauleApplicationArguments包装args
    ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
    // 准备容器环境
    ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
    // 配置需要ignore的bean
    configureIgnoreBeanInfo(environment);
    // 打印banner
    Banner printedBanner = printBanner(environment);
    // 创建容器上下文
    context = createApplicationContext();
    // 获取异常报告器
    exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                                                     new Class[] { ConfigurableApplicationContext.class },context);
    // 准备容器
    prepareContext(context, environment, listeners, applicationArguments, printedBanner);
    // 刷新容器
    refreshContext(context);
    // 完成刷新(什么也没干)
    afterRefresh(context, applicationArguments);
    stopWatch.stop();
    if (this.logStartupInfo) {
      new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
    }
    // 发布ApplicationStarted事件
    listeners.started(context);
    // 调用runner
    callRunners(context, applicationArguments);
  }
  catch (Throwable ex) {
    // 处理调用失败
    handleRunFailure(context, ex, exceptionReporters, listeners);
    throw new IllegalStateException(ex);
  }

  try {
    // 发布ApplicationReady事件
    listeners.running(context);
  }
  catch (Throwable ex) {
    handleRunFailure(context, ex, exceptionReporters, null);
    throw new IllegalStateException(ex);
  }
  return context;
}
```

### prepareEnvironment

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// 创建和配置环境
		ConfigurableEnvironment environment = getOrCreateEnvironment();
    // 配置环境
		configureEnvironment(environment, applicationArguments.getSourceArgs());
    // 发布environmentPrepared事件
		listeners.environmentPrepared(environment);
    // 绑定到spring容器
		bindToSpringApplication(environment);
		if (!this.isCustomEnvironment) {
			environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
					deduceEnvironmentClass());
		}
		ConfigurationPropertySources.attach(environment);
		return environment;
}
```

#### getOrCreateEnvironment

```java
private ConfigurableEnvironment getOrCreateEnvironment() {
  if (this.environment != null) {
    // 环境已经创建过
    return this.environment;
  }
  switch (this.webApplicationType) {
    case SERVLET:
      // 创建servlet环境
      return new StandardServletEnvironment();
    case REACTIVE:
      // 创建reactive环境
      return new StandardReactiveWebEnvironment();
    default:
      return new StandardEnvironment();
  }
}
```

#### configureEnvironment

```java
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
  if (this.addConversionService) {
    // 添加转换服务
    ConversionService conversionService = ApplicationConversionService.getSharedInstance();
    environment.setConversionService((ConfigurableConversionService) conversionService);
  }
  // 配置 propertySources
  configurePropertySources(environment, args);
  // 配置profiles
  configureProfiles(environment, args);
}
```

##### configurePropertySources

```java
protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
  MutablePropertySources sources = environment.getPropertySources();
  if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
   // 添加默认的属性
    sources.addLast(new MapPropertySource("defaultProperties", this.defaultProperties));
  }
  // 配置命令行属性
  if (this.addCommandLineProperties && args.length > 0) {
    String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
    if (sources.contains(name)) {
      PropertySource<?> source = sources.get(name);
      CompositePropertySource composite = new CompositePropertySource(name);
      composite.addPropertySource(
        new SimpleCommandLinePropertySource("springApplicationCommandLineArgs", args));
      composite.addPropertySource(source);
      sources.replace(name, composite);
    }
    else {
      sources.addFirst(new SimpleCommandLinePropertySource(args));
    }
  }
}
```

##### configureProfiles

```java
// 配置 配置文件
protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
  environment.getActiveProfiles(); 
  Set<String> profiles = new LinkedHashSet<>(this.additionalProfiles);
  profiles.addAll(Arrays.asList(environment.getActiveProfiles()));
  environment.setActiveProfiles(StringUtils.toStringArray(profiles));
}
```

#### bindToSpringApplication

```java
protected void bindToSpringApplication(ConfigurableEnvironment environment) {
  try {
   // 绑定到spring上下文
    Binder.get(environment).bind("spring.main", Bindable.ofInstance(this));
  }
  catch (Exception ex) {
    throw new IllegalStateException("Cannot bind to SpringApplication", ex);
  }
}
```

### createApplicationContext

```java
protected ConfigurableApplicationContext createApplicationContext() {
  Class<?> contextClass = this.applicationContextClass;
  if (contextClass == null) {
    try {
      switch (this.webApplicationType) {
        case SERVLET:
          // servlet容器 创建org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext容器
          contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
          break;
        case REACTIVE:
          // reactive容器 创建org.springframework.boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext容器
          contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
          break;
        default:
          // 默认创建org.springframework.context.annotation.AnnotationConfigApplicationContext
          contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
      }
    }
    catch (ClassNotFoundException ex) {
      throw new IllegalStateException(
        "Unable create a default ApplicationContext, " + "please specify an ApplicationContextClass",
        ex);
    }
  }
  // 实例化容器
  return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

### prepareContext

```java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
			SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
  // 容器添加 environment环境
  context.setEnvironment(environment);
  // 后期处理程序上下文 配置 beanNameGenerator, resourceLoader, conversionService
  postProcessApplicationContext(context);
  // 应用初始化器
  applyInitializers(context);
  // 发布 context初始化事件
  listeners.contextPrepared(context);
  if (this.logStartupInfo) {
    logStartupInfo(context.getParent() == null);
    logStartupProfileInfo(context);
  }
  // Add boot specific singleton beans
  ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
  // 注册单例bean
  beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
  if (printedBanner != null) {
    beanFactory.registerSingleton("springBootBanner", printedBanner);
  }
  if (beanFactory instanceof DefaultListableBeanFactory) {
    // 设置允许beanDefinition覆盖
    ((DefaultListableBeanFactory) beanFactory)
    .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
  }
  // Load the sources
  Set<Object> sources = getAllSources();
  Assert.notEmpty(sources, "Sources must not be empty");
  // 加载资源 beanNameGenerator rourceLoader environment
  load(context, sources.toArray(new Object[0]));
  // 发布contextLoaded事件
  listeners.contextLoaded(context);
}
```

### refreshContext

```java
private void refreshContext(ConfigurableApplicationContext context) {
  refresh(context);
  if (this.registerShutdownHook) {
    try {
      // 注册钩子程序在容器close的时候使用
      context.registerShutdownHook();
    }
    catch (AccessControlException ex) {
      // Not allowed in some environments.
    }
  }
}
```

#### refresh

```java
public void refresh() throws BeansException, IllegalStateException {
  synchronized (this.startupShutdownMonitor) {
    // 准备刷新容器
    prepareRefresh();

    // 获取bean工厂
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

    // 准备ioc容器
    prepareBeanFactory(beanFactory);

    try {
      // 允许在上下文子类中对bean工厂进行后处理.
      postProcessBeanFactory(beanFactory);

      // 调用在上下文中注册为Bean的工厂处理器.
      invokeBeanFactoryPostProcessors(beanFactory);

      // 注册拦截Bean创建的Bean处理器.
      registerBeanPostProcessors(beanFactory);

      // 初始化消息源.
      initMessageSource();

      // 初始化事件多播器.
      initApplicationEventMulticaster();

      // 在特定上下文子类中初始化其他特殊bean.
      onRefresh();

      // 检查侦听器bean并注册它们.
      registerListeners();

      // 实例化所有剩余的（非延迟初始化）单例.
      finishBeanFactoryInitialization(beanFactory);

      // 发布事件.
      finishRefresh();
    }

    catch (BeansException ex) {
      if (logger.isWarnEnabled()) {
        logger.warn("Exception encountered during context initialization - " +
                    "cancelling refresh attempt: " + ex);
      }

      // 销毁已创建的单例以避免资源悬空.
      destroyBeans();

      // Reset 'active' flag.
      cancelRefresh(ex);

      // Propagate exception to caller.
      throw ex;
    }

    finally {
      // 在Spring的核心中重置常见的自省缓存，因为
      //我们可能不再需要单例bean的元数据
      resetCommonCaches();
    }
  }
}
```

