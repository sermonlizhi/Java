## 一、Resource注解解析过程

@Resource和@Autowired处理的逻辑基本相同，都是先查找注入点，然后再根据注入点进行属性注入，所不同的是@Resource的解析是由CommonAnnotationBeanPostProcessor来完成的，@Autowired解析由AutowiredAnnotationBeanPostProcessor完成，下面介绍@Resource解析的全过程

#### 1.1 寻找注入点

寻找注入点的时机与@Autowired一致，都是在实例化后，调用所有实现了MergedBeanDefinitionPostProcessor接口的postProcessMergedBeanDefinition()方法，同样Resource注解的处理类CommonAnnotationBeanPostProcessor也实现了MergedBeanDefinitionPostProcessor接口，所以会调用该类的postProcessMergedBeanDefinition()方法来寻找注入点

方法的第一行代码会去调用InitDestroyAnnotationBeanPostProcessor的postProcessMergedBeanDefinition()来缓存和检查初始化方法和销毁方法

第二行代码去查找@Resource注解的元数据

```java
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
    super.postProcessMergedBeanDefinition(beanDefinition, beanType, beanName);
    InjectionMetadata metadata = findResourceMetadata(beanName, beanType, null);
    metadata.checkConfigMembers(beanDefinition);
}
```

根据beanName先去缓存中获取，如果缓存中没有或者InjectionMetadata中的目标class与当前beanType不符，则调用buildResourceMetadata来生成注解元数据

```java
private InjectionMetadata findResourceMetadata(String beanName, final Class<?> clazz, @Nullable PropertyValues pvs) {
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
                metadata = buildResourceMetadata(clazz);
                this.injectionMetadataCache.put(cacheKey, metadata);
            }
        }
    }
    return metadata;
}
```

循环遍历当前类和父类的属性和方法，判断是否有@Resource注解，如果有，则生成ResourceElement对象，ResourceElement继承自InjectionMetadata.InjectedElement，将ResourceElement对象对象放入到InjectedElement对象结合中，然后生成InjectionMetadata对象返回

对于静态属性和静态方法，@Autowired不会进行处理，但@Resource则直接抛出异常

```java
private InjectionMetadata buildResourceMetadata(final Class<?> clazz) {
    if (!AnnotationUtils.isCandidateClass(clazz, resourceAnnotationTypes)) {
        return InjectionMetadata.EMPTY;
    }

    List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
    Class<?> targetClass = clazz;

    do {
        final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();

        ReflectionUtils.doWithLocalFields(targetClass, field -> {
            ……
            else if (field.isAnnotationPresent(Resource.class)) {
                if (Modifier.isStatic(field.getModifiers())) {
                    throw new IllegalStateException("@Resource annotation is not supported on static fields");
                }
                if (!this.ignoredResourceTypes.contains(field.getType().getName())) {
                    currElements.add(new ResourceElement(field, field, null));
                }
            }
        });

        ReflectionUtils.doWithLocalMethods(targetClass, method -> {
            Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
            if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
                return;
            }
            ……
                else if (bridgedMethod.isAnnotationPresent(Resource.class)) {
                    if (Modifier.isStatic(method.getModifiers())) {
                        throw new IllegalStateException("@Resource annotation is not supported on static methods");
                    }
                    Class<?>[] paramTypes = method.getParameterTypes();
                    if (paramTypes.length != 1) {
                        throw new IllegalStateException("@Resource annotation requires a single-arg method: " + method);
                    }
                    if (!this.ignoredResourceTypes.contains(paramTypes[0].getName())) {
                        PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
                        currElements.add(new ResourceElement(method, bridgedMethod, pd));
                    }
                }
            }
        });

        elements.addAll(0, currElements);
        targetClass = targetClass.getSuperclass();
    }
    while (targetClass != null && targetClass != Object.class);

    return InjectionMetadata.forElements(elements, clazz);
}
```

ResourceElement的构造方法如下，@Resource注解可以通过name和type两个方法指定注入Bean的名字和类型，如果没有指定name，则使用默认的名称，使用参数名或set方法去掉set前缀当作name

如果指定了type属性，检查属性的类型或方法的第一个参数类型是否与指定的类型匹配，如果不匹配将抛出异常

```java
public ResourceElement(Member member, AnnotatedElement ae, @Nullable PropertyDescriptor pd) {
    super(member, pd);
    Resource resource = ae.getAnnotation(Resource.class);
    String resourceName = resource.name();
    Class<?> resourceType = resource.type();

    // 使用@Resource时没有指定具体的name，那么则用field的name，或setXxx()中的xxx
    this.isDefaultName = !StringUtils.hasLength(resourceName);
    if (this.isDefaultName) {
        resourceName = this.member.getName();
        if (this.member instanceof Method && resourceName.startsWith("set") && resourceName.length() > 3) {
            resourceName = Introspector.decapitalize(resourceName.substring(3));
        }
    }
    // 使用@Resource时指定了具体的name，进行占位符填充
    else if (embeddedValueResolver != null) {
        resourceName = embeddedValueResolver.resolveStringValue(resourceName);
    }

    // @Resource除开可以指定bean，还可以指定type，type默认为Object
    if (Object.class != resourceType) {
        // 如果指定了type，则验证一下和field的类型或set方法的第一个参数类型，是否和所指定的resourceType匹配
        checkResourceType(resourceType);
    }
    else {
        // No resource type specified... check field/method.
        resourceType = getResourceType();
    }
    ……
}
```

#### 1.2 属性注入

属性注入是在属性填充的时候过程中完成，调用InstantiationAwareBeanPostProcessor实现类的postProcessProperties()，@Resource注解的属性注入是在CommonAnnotationBeanPostProcessor的postProcessProperties()中完成

- ##### 开始注入

  获取已经缓存的注入点，然后调用InjectedElement的inject()，这个地方不同于@Autowired，Autowired注解的属性注入分别会调用AutowiredFieldElement和AutowiredMethodElement的inject()

  ```java
  public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
      InjectionMetadata metadata = findResourceMetadata(beanName, bean.getClass(), pvs);
      try {
          metadata.inject(bean, beanName, pvs);
      }
      catch (Throwable ex) {
          throw new BeanCreationException(beanName, "Injection of resource dependencies failed", ex);
      }
      return pvs;
  }
  ```

  ```java
  public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
     Collection<InjectedElement> checkedElements = this.checkedElements;
     Collection<InjectedElement> elementsToIterate =
           (checkedElements != null ? checkedElements : this.injectedElements);
     if (!elementsToIterate.isEmpty()) {
        // 遍历每个注入点进行依赖注入
        for (InjectedElement element : elementsToIterate) {
           element.inject(target, beanName, pvs);
        }
     }
  }
  ```

- ##### 获取注入值

  InjectedElement的inject()源码如下，直接调用getResourceToInject()获取依赖值进行注入

  ```java
  protected void inject(Object target, @Nullable String requestingBeanName, @Nullable PropertyValues pvs)
        throws Throwable {
  
     if (this.isField) {
        Field field = (Field) this.member;
        ReflectionUtils.makeAccessible(field);
        field.set(target, getResourceToInject(target, requestingBeanName));
     }
     else {
        if (checkPropertySkipping(pvs)) {
           return;
        }
        try {
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

  如果属性或方法标注了@Lazy注解，则生成一个代理对象，否则调用getResource()获取注入对象

  ```java
  protected Object getResourceToInject(Object target, @Nullable String requestingBeanName) {
     return (this.lazyLookup ? buildLazyResourceProxy(this, requestingBeanName) :
           getResource(this, requestingBeanName));
  }
  ```

  getResource()源码如下，调用核心的autowireResource()获取bean对象

  ```java
  protected Object getResource(LookupElement element, @Nullable String requestingBeanName)
        throws NoSuchBeanDefinitionException {
  	……
     // 根据LookupElement从BeanFactory找到适合的bean对象
     return autowireResource(this.resourceFactory, element, requestingBeanName);
  }
  ```

  如果当前的beanFactory属于AutowireCapableBeanFactory，如果根据名称找不到bean对象时，还可以调用resolveDependency()根据type获取或生成bean对象

  @Resource注解查找bean对象时，先判断是否使用默认名，如果用默认名，则先根据默认名判断BeanFactory是否有，如果有的话直接调用getBean()获取bean对象，如果没有，再调用resolveDependency()根据类型去查找或生成bean对象，resolveDependency()是@Autowired解析的核心方法

  如果指定了名称，则直接调用getBean()方法去查找bean对象

  如果找到了bean对象，则将它的name缓存在autowiredBeanNames中，最后调用beanFactory.registerDependentBean()缓存beanName之间的依赖关系，当前注入的bean对象的被哪些bean依赖，缓存的是beanNaem，而不是bean对象

  ```java
  protected Object autowireResource(BeanFactory factory, LookupElement element, @Nullable String requestingBeanName)
        throws NoSuchBeanDefinitionException {
  
     Object resource;
     Set<String> autowiredBeanNames;
     String name = element.name;
  
     if (factory instanceof AutowireCapableBeanFactory) {
        AutowireCapableBeanFactory beanFactory = (AutowireCapableBeanFactory) factory;
        DependencyDescriptor descriptor = element.getDependencyDescriptor();
  
        // 假设@Resource中没有指定name，并且field的name或setXxx()的xxx不存在对应的bean，那么则根据field类型或方法参数类型从BeanFactory去找
        if (this.fallbackToDefaultTypeMatch && element.isDefaultName && !factory.containsBean(name)) {
           autowiredBeanNames = new LinkedHashSet<>();
           resource = beanFactory.resolveDependency(descriptor, requestingBeanName, autowiredBeanNames, null);
           if (resource == null) {
              throw new NoSuchBeanDefinitionException(element.getLookupType(), "No resolvable resource object");
           }
        }
        else {
           resource = beanFactory.resolveBeanByName(name, descriptor);
           autowiredBeanNames = Collections.singleton(name);
        }
     }
     else {
        resource = factory.getBean(name, element.lookupType);
        autowiredBeanNames = Collections.singleton(name);
     }
  
     if (factory instanceof ConfigurableBeanFactory) {
        ConfigurableBeanFactory beanFactory = (ConfigurableBeanFactory) factory;
        for (String autowiredBeanName : autowiredBeanNames) {
           if (requestingBeanName != null && beanFactory.containsBean(autowiredBeanName)) {
              beanFactory.registerDependentBean(autowiredBeanName, requestingBeanName);
           }
        }
     }
  
     return resource;
  }
  ```

#### 1.3 与@Autowired区别

1、性能上两者没有任何区别，@Resource是Jdk提供了注解，Spring对其进行了具体实现，在获取依赖注入值的过程中，@Autowired先根据类型来查找，然后进行Qualifier筛选等层层筛选，而@Resource先根据名城查找，如果使用的是默认名，并且beanFactory根据名称找不到的时候，才会调用与@Autowired一样的逻辑根据类型来找

2、如果@Resource指定了名称，则只会根据名称来查找bean对象，找不到直接抛异常
