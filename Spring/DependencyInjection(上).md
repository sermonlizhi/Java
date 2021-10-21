## 一、依赖注入方式

在创建Bean过程中，当完成了Bean的实例化以后，会对Bean进行属性填充，属性填充的过程中完成依赖注入

依赖注入主要有两种方式

- 手动注入
- 自动注入

#### 1.1 手动注入

手动注入通过xml配置的方式来完成，包含set方法注入和构造方法注入两种方式：

- set方法注入

  通过set方法注入时，对应的属性必须要有set方法，同时set方法的方法名必须满足JavaBean的规范

  ```xml
  <!--set注入-->
  <bean id="userService" class="com.lizhi.service.UserService" >
     <property name="orderService" ref="orderService"/>
  </bean>
  ```

- 构造方法注入

  xml中使用构造方法注入时，需要指定构造方法的参数下标

  ```xml
  <bean id="userService" class="com.lizhi.service.UserService" >
     <constructor-arg index="0" ref="orderService"/>
  </bean>
  ```

#### 1.2 自动注入

自动注入主要通过@Autowired和autowire属性来实现，其中autowire属性的使用场景有如下两种方式

- xml的方式

  xml方式中，autowire属性定义了五种注入方式：no、byName、byType、construct、default

  ```xml
  <bean id="userService" class="com.lizhi.service.UserService" autowire="byName"/>
  ```

- @Bean方式

  @Bean注解中，autowire属性只定义了三种注入方式：NO、BY_NAME、BY_TYPE，默认为NO

  这种方式现在已经不推荐使用了

  ```java
  @Bean(autowire = BY_NAME)
  public UserService userService(){
     return new UserService();
  }
  ```



## 二、依赖注入的基本过程

#### 2.1 autowire属性注入

处理依赖注入的时候，最先解决的就是通过autowire属性指定注入方式的这种依赖注入，源码如下：

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
        // MutablePropertyValues是PropertyValues具体的实现类
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
        // Add property values based on autowire by name if applicable.
        if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }
        // Add property values based on autowire by type if applicable.
        if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }
        pvs = newPvs;
    }
```

- 获取自定义属性值

  首先从RootBeanDefinition中去拿由开发人员自己添加的属性值，开发人员可以在MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition方法中添加属性值

  BeanDefinition中的属性值存放在MutablePropertyValues缓存中，MutablePropertyValues实例的propertyValueList存放了具体的属性相关信息，包括属性名、属性值等等

  ```java
  public class LzBeanPostProcessor implements MergedBeanDefinitionPostProcessor {
      @Override
      public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
          beanDefinition.getPropertyValues().add("user",new User());
      }
  }
  ```

  MutablePropertyValues类定义如下：

  ```java
  public class MutablePropertyValues implements PropertyValues, Serializable {
  
  	private final List<PropertyValue> propertyValueList;
      ……
  }
  ```

  PropertyValue类定义如下

  ```java
  public class PropertyValue extends BeanMetadataAttributeAccessor implements Serializable {
  
      private final String name;
  
      @Nullable
      private final Object value;
      ……
  }
  ```

- 判断注入方式

  调用getResolvedAutowireMode()方法的获取BeanDefinition中指定的注入方式，如果是通过BY_NAME或BY_TYPE方式注入，则会调用对应的方法，获取对应的参数与值，但并不是直接对属性赋值，而是先将其缓存到PropertyValues的属性列表中

- 按属性名称注入

  如果是按名称进行注入，则调用autowireByName()解析属性

  ```java
  protected void autowireByName(
        String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
  
     // 当前Bean中能进行自动注入的属性名
     String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
     // 遍历每个属性名，并去获取Bean对象，并设置到pvs中
     for (String propertyName : propertyNames) {
        if (containsBean(propertyName)) {
           Object bean = getBean(propertyName);
           pvs.add(propertyName, bean);
           // 记录一下propertyName对应的Bean被beanName给依赖了
           registerDependentBean(propertyName, beanName);
        }
     }
  }
  ```

  首先会调用unsatisfiedNonSimpleProperties()判断哪些属性可以进行注入，从当前实例获取所有的属性描述实例PropertyDescriptor对象，该对象记录了属性的get和set方法等信息

  如果当前属性有set方法，并且已经定义的属性中没有该属性，并且属性类型不是简单类型，isSimpleProperty()判断是否是简单类型，简单类型如：Enum、Number、CharSequence等

  ```java
  protected String[] unsatisfiedNonSimpleProperties(AbstractBeanDefinition mbd, BeanWrapper bw) {
     Set<String> result = new TreeSet<>();
     PropertyValues pvs = mbd.getPropertyValues();
     PropertyDescriptor[] pds = bw.getPropertyDescriptors();
  
     for (PropertyDescriptor pd : pds) {
        if (pd.getWriteMethod() != null && !isExcludedFromDependencyCheck(pd) && !pvs.contains(pd.getName()) &&
              !BeanUtils.isSimpleProperty(pd.getPropertyType())) {
           result.add(pd.getName());
        }
     }
     return StringUtils.toStringArray(result);
  }
  ```

  得到需要注入的属性的属性名之后，如上面代码所示，遍历属性名，调用getBean()去获取属性名实例作为属性值，这个时候并没有对属性进行赋值，只是把属性名与值缓存到了MutablePropertyValues的实例pvs中

- 按属性类型注入

  如果是按类型注入，则调用autowireByType()，判断哪些属性需要注入的逻辑同上面按属性名称注入，然后调用resolveDependency()方法根据类型获取Bean实例，然后进行缓存，该方法的具体实现，会在依赖注入的其他章节详细讲到

  ```java
  protected void autowireByType(
      String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
  
      Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
  
      // 当前Bean中能进行自动注入的属性名
      String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
      for (String propertyName : propertyNames) {
          PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
          if (Object.class != pd.getPropertyType()) {
              MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
              boolean eager = !(bw.getWrappedInstance() instanceof PriorityOrdered);
              DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
              // 根据类型找到的结果
              Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, 
              if (autowiredArgument != null) {
                  pvs.add(propertyName, autowiredArgument);
              }
              ……
          }
      }
  }
  ```

#### 2.2 @Autowired注入

执行完前面autowired属性注入的逻辑之后，接下来将处理@Autowired、@Value、@Inject、@Resourec这些注解，并直接给属性进行赋值

获取实现了InstantiationAwareBeanPostProcessor接口的所有Bean，该接口中提供了postProcessProperties()方法，用于给属性赋值，遍历所有该类Bean，调用postProcessProperties实现属性赋值

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    ……
    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

    PropertyDescriptor[] filteredPds = null;
    if (hasInstAwareBpps) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }
        for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
            // 这里会调用AutowiredAnnotationBeanPostProcessor的postProcessProperties()方法，会直接给对象中的属性赋值
            // AutowiredAnnotationBeanPostProcessor内部并不会处理pvs，直接返回了
            PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
                if (filteredPds == null) {
                    filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                }
                pvsToUse = bp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                if (pvsToUse == null) {
                    return;
                }
            }
            pvs = pvsToUse;
        }
    }
    ……
}
```

下面以AutowiredAnnotationBeanPostProcessor为例，详细介绍属性赋值

AutowiredAnnotationBeanPostProcessor不仅实现了InstantiationAwareBeanPostProcessor接口，还实现了MergedBeanDefinitionPostProcessor，并且在MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition方法的实现中，做了非常重要的工作

- 寻找注入点

  postProcessMergedBeanDefinition()的调用是在实例化后，属性填充前调用的，该方法中主要实现了寻找注入点的逻辑，即判断哪些方法或属性需要注入

  ```java
  public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
     InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
     metadata.checkConfigMembers(beanDefinition);
  }
  ```

  调用findAutowiringMetadata()寻找注入点，首先从缓存中取出所有注入点，然后将pvs中的属性从注入点中移除，pvs中属性的赋值会有其他逻辑去做，然后调用buildAutowiringMetadata()方法去真正解析注入点

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
              // 解析注入点并缓存
              metadata = buildAutowiringMetadata(clazz);
              this.injectionMetadataCache.put(cacheKey, metadata);
           }
        }
     }
     return metadata;
  }
  ```

  解析注入点，从当前类开始，遍历类中所有属性和方法，调用findAutowiredAnnotation()是否有注解，但是对于static修饰的属性和方法不是注入点，在遍历方法的时候，需要调用findBridgedMethod()判断当前方法是否是桥接方法，桥接方法不会生成注入点；然后将生成的注入点全部缓存起来，后面注入属性的时候会用到

  ```java
  private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
      do {
          final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();
  
          // 遍历targetClass中的所有Field
          ReflectionUtils.doWithLocalFields(targetClass, field -> {
              // field上是否存在@Autowired、@Value、@Inject中的其中一个
              MergedAnnotation<?> ann = findAutowiredAnnotation(field);
              if (ann != null) {
                  // static filed不是注入点，不会进行自动注入
                  if (Modifier.isStatic(field.getModifiers())) {
                      return;
                  }
  
                  // 构造注入点
                  boolean required = determineRequiredStatus(ann);
                  currElements.add(new AutowiredFieldElement(field, required));
              }
          });
  
          // 遍历targetClass中的所有Method
          ReflectionUtils.doWithLocalMethods(targetClass, method -> {
  
              Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
              if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
                  return;
              }
              // method上是否存在@Autowired、@Value、@Inject中的其中一个
              MergedAnnotation<?> ann = findAutowiredAnnotation(bridgedMethod);
              if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
                  // static method不是注入点，不会进行自动注入
                  if (Modifier.isStatic(method.getModifiers())) {
                      return;
                  }
                  boolean required = determineRequiredStatus(ann);
                  PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
                  currElements.add(new AutowiredMethodElement(method, required, pd));
              }
          });
  
          elements.addAll(0, currElements);
          targetClass = targetClass.getSuperclass();
      }
      while (targetClass != null && targetClass != Object.class);
  
      return InjectionMetadata.forElements(elements, clazz);
  }
  ```

  findAutowiredAnnotation判断是否有@Autowired、@Value、@Inject中的其中一个，autowiredAnnotationTypes是一个注解缓存，在AutowiredAnnotationBeanPostProcessor实例化的时候，会给该缓存添加Autowired、Value和Inject这三个注解

  ```java
  private MergedAnnotation<?> findAutowiredAnnotation(AccessibleObject ao) {
     MergedAnnotations annotations = MergedAnnotations.from(ao);
     for (Class<? extends Annotation> type : this.autowiredAnnotationTypes) {
        MergedAnnotation<?> annotation = annotations.get(type);
        if (annotation.isPresent()) {
           return annotation;
        }
     }
     return null;
  }
  ```

  ```java
  private final Set<Class<? extends Annotation>> autowiredAnnotationTypes = new LinkedHashSet<>(4);
  // 构造方法
  public AutowiredAnnotationBeanPostProcessor() {
      this.autowiredAnnotationTypes.add(Autowired.class);
      this.autowiredAnnotationTypes.add(Value.class);
      this.autowiredAnnotationTypes.add((Class<? extends Annotation>)
                                            ClassUtils.forName("javax.inject.Inject", AutowiredAnnotationBeanPostProcessor.class.getClassLoader()));
  }
  ```

  findBridgedMethod()判断是否为桥接方法，对于某个类，如果它实现了泛型接口的某个方法，则在它的字节码文件中，会存在两个方法，一个真正的方法，一个是桥接方法

  ```java
  public interface BridgeInterface <T>{
  	void test(T t);
  }
  
  @Component
  public class BridgeInterfaceImpl implements BridgeInterface<UserService>{
  	@Override
  	@Autowired
  	public void test(UserService userService) {
  
  	}
  }
  ```

  查看它的反编译后的字节码文件，发现有两个test方法，其中一个是桥接方法，对于桥接方法，不会生成注入点

  ```java
  public test(Lcom/lizhi/service/UserService;)V
  public synthetic bridge test(Ljava/lang/Object;)V
  ```

- 属性注入

  属性注入是在AutowiredAnnotationBeanPostProcessor.postProcessProperties()中完成的

  同样的调用findAutowiringMetadata()去获取注入点，因为前面已经寻找过注入点且缓存了，那么这一次可以直接从缓存中取出所有注入点，然后进行注入

  ```java
  	public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
  		// 找注入点（所有被@Autowired注解了的Field或Method）
  		InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
  		try {
  			metadata.inject(bean, beanName, pvs);
  		}
  		return pvs;
  	}
  ```

  注入方法的实现如下，遍历每个注入点，然后调用注入点的inject()进行注入

  如果是属性注入点，则调用AutowiredFieldElement.inject()方法类进行注入，主要通过resolveFieldValue()从BeanFactory中找到匹配的Bean实例，resolveFieldValue()中的核心方法是beanFactory.resolveDependency()
  
  ```java
  protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
  
      Field field = (Field) this.member;
      Object value;
      if (this.cached) {
          // 对于原型Bean，第一次创建的时候，也找注入点，然后进行注入，此时cached为false，注入完了之后cached为true
          // 第二次创建的时候，先找注入点（此时会拿到缓存好的注入点），也就是AutowiredFieldElement对象，此时cache为true，也就进到此处了
          // 注入点内并没有缓存被注入的具体Bean对象，而是beanName，这样就能保证注入到不同的原型Bean对象
          try {
              value = resolvedCachedArgument(beanName, this.cachedFieldValue);
          }
          catch (NoSuchBeanDefinitionException ex) {
              // Unexpected removal of target bean for cached argument -> re-resolve
              value = resolveFieldValue(field, bean, beanName);
          }
      }
      else {
          // 根据filed从BeanFactory中查到的匹配的Bean对象
          value = resolveFieldValue(field, bean, beanName);
      }
  
      // 反射给filed赋值
      ……
  }
  ```

  如果是方法注入点，则调用AutowiredMethodElement.inject()进行注入，方法注入的逻辑要比属性注入复杂很多，resolveMethodArguments()中的核心方法也是beanFactory.resolveDependency()
  
  ```java
  protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
     // 如果pvs中已经有当前注入点的值了，则跳过注入
     if (checkPropertySkipping(pvs)) {
        return;
     }
     Method method = (Method) this.member;
     Object[] arguments;
     if (this.cached) {
        try {
           arguments = resolveCachedArguments(beanName);
        }
        catch (NoSuchBeanDefinitionException ex) {
           // Unexpected removal of target bean for cached argument -> re-resolve
           arguments = resolveMethodArguments(method, bean, beanName);
        }
     }
     else {
        arguments = resolveMethodArguments(method, bean, beanName);
     }
     if (arguments != null) {
        ReflectionUtils.makeAccessible(method);
           method.invoke(bean, arguments);
     }
  }
  ```
  
  **无论是属性注入点还是方法注入点，在进行注入的时候，它们最核心的代码都是BeanFactory的resolveDependency()，调用该方法来获取注入的属性值，即Bean实例对象，下一章节会对依赖注入的核心方法，如果获取依赖对象作详细介绍**
  
  `@Resource的注入是在CommonAnnotationBeanPostProcessor类中,寻找注入点以及属性注入同上`

## 三、设置BeanDefinition中的属性值

在前面的属性注入中，对于RootBeanDefinition中的定义的属性MutablePropertyValues却没有赋值，而对这部分属性的赋值操作，是在完成@Autowired和@@Resource这类注解的属性注入后进行

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    ……
    if (pvs != null) {
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```

调用applyPropertyValues()方法来对进行属性赋值