# Autowired原理解析

autowired注解主要是将我们主要的bean从ioc容器中注入到我们的引用类中,Bean的注入其实是在spring刷新ioc容器的时候调用的,其主要实现是在AbstractApplicationContext的refresh方法中,该方法中的的finishBeanFactoryInitialization来讲不是懒加载的bean进行初始化,Autowired是可以添加在属性/构造方法/Setter方法/普通方法上,它主要的实现逻辑在AutowiredAnnotationBeanPostProcessor中Autowired注解是在spring的org.springframework.beans.factory.annotation包下面

## postProcessProperties

```java
// 后处理属性
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
  // 获取Autowired元信息
  InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
  try {
    // 注入
    metadata.inject(bean, beanName, pvs);
  }
  catch (BeanCreationException ex) {
    throw ex;
  }
  catch (Throwable ex) {
    throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
  }
  return pvs;
}
```



### findAutowiringMetadata

```java
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {
  // Fall back to class name as cache key, for backwards compatibility with custom callers.
  String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
  // Quick check on the concurrent map first, with minimal locking.
  InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
  if (InjectionMetadata.needsRefresh(metadata, clazz)) {
    synchronized (this.injectionMetadataCache) {
      metadata = this.injectionMetadataCache.get(cacheKey);
      if (InjectionMetadata.needsRefresh(metadata, clazz)) {
        if (metadata != null) {
          metadata.clear(pvs);
        }
        // 构建Autowired元信息
        metadata = buildAutowiringMetadata(clazz);
        this.injectionMetadataCache.put(cacheKey, metadata);
      }
    }
  }
  return metadata;
}
```

#### buildAutowiringMetadata

```java
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
  List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
  Class<?> targetClass = clazz;

  do {
    final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();
		// 通过反射获取目标类的所有field,
    ReflectionUtils.doWithLocalFields(targetClass, field -> {
      // 判断field是否被Autowired注解修饰
      AnnotationAttributes ann = findAutowiredAnnotation(field);
      if (ann != null) {
        // 判断field是否被static修饰
        if (Modifier.isStatic(field.getModifiers())) {	
          if (logger.isInfoEnabled()) {
            logger.info("Autowired annotation is not supported on static fields: " + field);
          }
          return;
        }
        // 判断是否指定了required属性
        boolean required = determineRequiredStatus(ann);
        currElements.add(new AutowiredFieldElement(field, required));
      }
    });

    // 通过反射获取目标类的method
    ReflectionUtils.doWithLocalMethods(targetClass, method -> {
      Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
      if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
        return;
      }
      // 查看方法是否被Autowired修饰
      AnnotationAttributes ann = findAutowiredAnnotation(bridgedMethod);
      if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
        // 静态方法
        if (Modifier.isStatic(method.getModifiers())) {
          if (logger.isInfoEnabled()) {
            logger.info("Autowired annotation is not supported on static methods: " + method);
          }
          return;
        }
        // 无参方法
        if (method.getParameterCount() == 0) {
          if (logger.isInfoEnabled()) {
            logger.info("Autowired annotation should only be used on methods with parameters: " +
                        method);
          }
        }
        // 判断是否有required属性
        boolean required = determineRequiredStatus(ann);
        // 获取属性描述符
        PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
        currElements.add(new AutowiredMethodElement(method, required, pd));
      }
    });
		// 被Autowired修饰的可能不止一个，所有都添加到currElements这个容器中一起处理
    elements.addAll(0, currElements);
    targetClass = targetClass.getSuperclass();
  }
  while (targetClass != null && targetClass != Object.class);

  return new InjectionMetadata(clazz, elements);
}
```

### inject(target, beanName, pvs)

```java
public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
  Collection<InjectedElement> checkedElements = this.checkedElements;
  Collection<InjectedElement> elementsToIterate =
    (checkedElements != null ? checkedElements : this.injectedElements);
  if (!elementsToIterate.isEmpty()) {
    for (InjectedElement element : elementsToIterate) {
      if (logger.isTraceEnabled()) {
        logger.trace("Processing injected element of bean '" + beanName + "': " + element);
      }
      // 循环注入
      element.inject(target, beanName, pvs);
    }
  }
}
```

#### inject()

```java
protected void inject(Object target, @Nullable String requestingBeanName, @Nullable PropertyValues pvs)
  throws Throwable {

  if (this.isField) {
    // 属性的话 设置为属性可见并且给属性赋值
    Field field = (Field) this.member;
    ReflectionUtils.makeAccessible(field);
    // 获取资源并且赋值给field
    field.set(target, getResourceToInject(target, requestingBeanName));
  }
  else {
    if (checkPropertySkipping(pvs)) {
      return;
    }
    try {
      // 方法的话 先设置可见 然后调用此方法
      Method method = (Method) this.member;
      ReflectionUtils.makeAccessible(method);
      method.invoke(target, getResourceToInject(target, requestingBeanName));
    }
    catch (InvocationTargetException ex) {
      throw ex.getTargetException();
    }
  }
}
```

##### getResourceToInject()

```
// 具体的实现在ResourceElement中
protected Object getResourceToInject(Object target, @Nullable String requestingBeanName) {
   //  判断是不是懒加载的
   return (this.lazyLookup ? 
         // 构建懒加载资源的代理
         buildLazyResourceProxy(this, requestingBeanName) :
         // 获取资源
         getResource(this, requestingBeanName));
}
```



