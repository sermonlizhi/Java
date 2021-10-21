在AbstractBeanFactory类的doGetBean()中，都是调用AbstractAutowireCapableBeanFactory类的createBean()来创建Bean实例，该方法参数如下：

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
```

第一个参数为beanName，第二个为beanName对应的RootBeanDefinition，第三个参数为构造器的参数列表，实例化的时候需要参数列表推断使用哪一个构造器来进行实例化

## 1、加载beanClass

在扫描指定包下面的class文件生成BeanDefinition的时候，beanDefinition中的beanClass属性暂时存放的是类的名称(全称)，而不是加载后的Class对象，但是在实例化Bean对象之前，JVM需要加载将class文件加载进来，并用beanDefinition中的beanClass存储Class对象

```java
RootBeanDefinition mbdToUse = mbd;
// 马上就要实例化Bean了，确保beanClass被加载了
Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
    mbdToUse = new RootBeanDefinition(mbd);
    mbdToUse.setBeanClass(resolvedClass);
}
```

下面看resolveBeanClass(mbd, beanName)是如何获取Class对象的，如果该RootBeanDefinition对应的Class对象已经加载，则直接返回该对象，否则调用doResolveBeanClass(mbd, typesToMatch)方法来加载Class对象

```java
protected Class<?> resolveBeanClass(RootBeanDefinition mbd, String beanName, Class<?>... typesToMatch)
    throws CannotLoadBeanClassException {

    try {
        // 如果beanClass被加载了
        if (mbd.hasBeanClass()) {
            return mbd.getBeanClass();
        }
        ……		
            // 如果beanClass没有被加载
            return doResolveBeanClass(mbd, typesToMatch);
    }
}
```

看doResolveBeanClass是如何实现的

```java
private Class<?> doResolveBeanClass(RootBeanDefinition mbd, Class<?>... typesToMatch)
    throws ClassNotFoundException {

    // 获取类加载器
    ClassLoader beanClassLoader = getBeanClassLoader();
    ClassLoader dynamicLoader = beanClassLoader;
    boolean freshResolve = false;
    if (!ObjectUtils.isEmpty(typesToMatch)) {
        // When just doing type checks (i.e. not creating an actual instance yet),
        // use the specified temporary class loader (e.g. in a weaving scenario).
        ClassLoader tempClassLoader = getTempClassLoader();
        if (tempClassLoader != null) {
            dynamicLoader = tempClassLoader;
            freshResolve = true;
            if (tempClassLoader instanceof DecoratingClassLoader) {
                DecoratingClassLoader dcl = (DecoratingClassLoader) tempClassLoader;
                for (Class<?> typeToMatch : typesToMatch) {
                    dcl.excludeClass(typeToMatch.getName());
                }
            }
        }
    }
```

- 首先获取类加载器，getBeanClassLoader()会调用ClassUtils类中的getDefaultClassLoader()获取类加载器，获取类加载的代码如下：

  首先获取当前线程指定上下文加载器，可以在Spring容器启动之前，设置当前线程(主线程)上下文的类加载器

  ```java
  Thread.currentThread().setContextClassLoader(ClassLoader.getSystemClassLoader());
  ```

  如果没有指定，则获取当前类ClassUtils的类加载器，如果当前类是通过bootstrap加载的，则获取系统类加载器

  ```java
  public static ClassLoader getDefaultClassLoader() {
      ClassLoader cl = null;
  
      // 优先获取线程中的类加载器
      cl = Thread.currentThread().getContextClassLoader();
  
      // 线程中类加载器为null的情况下，获取加载ClassUtils类的类加载器
      if (cl == null) {
          // No thread context class loader -> use class loader of this class.
          cl = ClassUtils.class.getClassLoader();
          if (cl == null) {
              // getClassLoader() returning null indicates the bootstrap ClassLoader
              // 假如ClassUtils是被Bootstrap类加载器加载的，则获取系统类加载器
              cl = ClassLoader.getSystemClassLoader();
          }
      }
      return cl;
  }
  ```

- 判断beanClassName是否是EL表达式

  getBeanClassName()方法返回的可能是类加载的名字，也可能只是类名，如果beanClass是一个Spring的EL表达式，则evaluateBeanDefinitionString可以将其解析，返回一个Class对象或类名

  ```java
  String className = mbd.getBeanClassName();
  if (className != null) {
      // 解析Spring表达式，有可能直接返回了一个Class对象
      Object evaluated = evaluateBeanDefinitionString(className, mbd);
      if (!className.equals(evaluated)) {
          // A dynamically resolved expression, supported as of 4.2...
          if (evaluated instanceof Class) {
              return (Class<?>) evaluated;
          }
          else if (evaluated instanceof String) {
              className = (String) evaluated;
              freshResolve = true;
          }
      }
      if (freshResolve) {
          if (dynamicLoader != null) {
             return dynamicLoader.loadClass(className);
          }
          return ClassUtils.forName(className, dynamicLoader);
      }
  }
  ```

- 解析生成beanClass

  如果前面的条件都无法得到Class对象，则调用RootBeanDefinition的resolveBeanClass方法

  ```java
  @Nullable
  private Class<?> doResolveBeanClass(RootBeanDefinition mbd, Class<?>... typesToMatch)
      throws ClassNotFoundException {
      ……
      return mbd.resolveBeanClass(beanClassLoader);
  }
  ```

  resolveBeanClass()中调用 ClassUtils.forName生成对应Class对象，然后将Class对象存储在RootBeanDefinition的beanClass属性中

  ```java
  public Class<?> resolveBeanClass(@Nullable ClassLoader classLoader) throws ClassNotFoundException {
      String className = getBeanClassName();
      if (className == null) {
          return null;
      }
      Class<?> resolvedClass = ClassUtils.forName(className, classLoader);
      this.beanClass = resolvedClass;
      return resolvedClass;
  }
  ```

## 2、实例化前

在实例化前操作的前面，会调用prepareMethodOverrides()处理带有@Lookup注解的方法，然后再进行实例化前的操作

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    throws BeanCreationException {
    ……
    mbdToUse.prepareMethodOverrides();
    ……
}
```

在对Bean进行实例化之前，调用resolveBeforeInstantiation()执行初始化前的操作，如果初始化前的操作直接就生成了Bean对象，则直接返回，没必要再执行后面的实例化操作

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    throws BeanCreationException {
    ……
    // 实例化前
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
        return bean;
    }
    ……
}
```

resolveBeforeInstantiation()中，hasInstantiationAwareBeanPostProcessors()会判断当前Spring容器的Bean中是否有实现了InstantiationAwareBeanPostProcessor接口的Bean

```java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    Object bean = null;
    if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
        // Make sure bean class is actually resolved at this point.
        // synthetic表示合成，如果某些Bean式合成的，那么则不会经过BeanPostProcessor的处理
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            Class<?> targetType = determineTargetType(beanName, mbd);
            if (targetType != null) {
                bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                if (bean != null) {
                    bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                }
            }
        }
        mbd.beforeInstantiationResolved = (bean != null);
    }
    return bean;
}
```

- 判断是否有Bean实现了InstantiationAwareBeanPostProcessor接口

  ```java
  protected boolean hasInstantiationAwareBeanPostProcessors() {
      return !getBeanPostProcessorCache().instantiationAware.isEmpty();
  }
  ```

  在讲清楚InstantiationAwareBeanPostProcessor之前，需要先了解一下AbstractBeanFactory中的静态内部类BeanPostProcessorCache，源码如下：

  ```java
  static class BeanPostProcessorCache {
  
      final List<InstantiationAwareBeanPostProcessor> instantiationAware = new ArrayList<>();
  
      final List<SmartInstantiationAwareBeanPostProcessor> smartInstantiationAware = new ArrayList<>();
  
      final List<DestructionAwareBeanPostProcessor> destructionAware = new ArrayList<>();
  
      final List<MergedBeanDefinitionPostProcessor> mergedDefinition = new ArrayList<>();
  }
  ```

  BeanPostProcessCache中分别缓存实现了InstantiationAwareBeanPostProcessor、SmartInstantiationAwareBeanPostProcessor、DestructionAwareBeanPostProcessor和MergedBeanDefinitionPostProcessor接口的Bean对象，而这四个接口都继承自BeanPostProcessor接口，BeanPostProcessor接口定义了postProcessBeforeInitialization()和postProcessAfterInitialization两个方法，即初始化前和初始化需要调用的方法，在这两个方法的基础上，上述四个接口也定义各自的方法，源码如下

  ```java
  public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {
      // 实例化前调用
      default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
          return null;
      }
      
      // 实例化后调用
      default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
          return true;
      }
      
      //属性填充
      default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
          throws BeansException {
  
          return null;
      }
  }
  ```

  ```java
  public interface SmartInstantiationAwareBeanPostProcessor extends InstantiationAwareBeanPostProcessor {
      default Class<?> predictBeanType(Class<?> beanClass, String beanName) throws BeansException {
          return null;
      }
      default Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName)
          throws BeansException {
  
          return null;
      }
      default Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
          return bean;
      }
  }
  ```

  ```java
  public interface DestructionAwareBeanPostProcessor extends BeanPostProcessor {
      void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException;
      default boolean requiresDestruction(Object bean) {
          return true;
      }
  }
  ```

  ```java
  public interface MergedBeanDefinitionPostProcessor extends BeanPostProcessor {
      void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);
      default void resetBeanDefinition(String beanName) {
      }
  
  }
  ```

  下面介绍getBeanPostProcessorCache()方法的实现

  AbstractBeanFactory类的beanPostProcessorCache属性是BeanPostProcessorCache类的实例，缓存各种实现了BeanPostPocessor的Bean对象，如果当前缓存为空，就从beanPostProcessors中取出所有实现了BeanPostProcessor接口的Bean实例，然后将它们进行细分之后，进行缓存

  ```java
  BeanPostProcessorCache getBeanPostProcessorCache() {
      BeanPostProcessorCache bpCache = this.beanPostProcessorCache;
      if (bpCache == null) {
          bpCache = new BeanPostProcessorCache();
          for (BeanPostProcessor bp : this.beanPostProcessors) {
              if (bp instanceof InstantiationAwareBeanPostProcessor) {
                  bpCache.instantiationAware.add((InstantiationAwareBeanPostProcessor) bp);
                  if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                      bpCache.smartInstantiationAware.add((SmartInstantiationAwareBeanPostProcessor) bp);
                  }
              }
              if (bp instanceof DestructionAwareBeanPostProcessor) {
                  bpCache.destructionAware.add((DestructionAwareBeanPostProcessor) bp);
              }
              if (bp instanceof MergedBeanDefinitionPostProcessor) {
                  bpCache.mergedDefinition.add((MergedBeanDefinitionPostProcessor) bp);
              }
          }
          this.beanPostProcessorCache = bpCache;
      }
      return bpCache;
  }
  ```

- 调用实例化前方法

  上面已经详细介绍了beanPostProcessorCache，现在再回到resolveBeforeInstantiation()方法，如果有实现了InstantiationAwareBeanPostProcessor接口的Bean，调用determineTargetType()推断出BeanDefinition所代表的类型

  ```java
  if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      Class<?> targetType = determineTargetType(beanName, mbd);
      if (targetType != null) {
          bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
          if (bean != null) {
              bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
          }
      }
  }
  ```

  然后调用applyBeanPostProcessorsBeforeInstantiation()执行实例化前方法，beanPostProcessorCache可能有多个实现了InstantiationAwareBeanPostProcessor接口的Bean对象，则遍历调用这些对象的postProcessBeforeInstantiation()方法，如果某个对象调用该方法时返回了一个对象，则直接返回该对象

  ```java
  protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
     for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
        Object result = bp.postProcessBeforeInstantiation(beanClass, beanName);
        if (result != null) {
           return result;
        }
     }
     return null;
  }
  ```

- 如果调用实例化前方法产生了对象，则调用初始化后方法

  在调用实例化前方法时，如果返回了不为空的对象，则调用applyBeanPostProcessorsAfterInitialization()，遍历所有实现了BeanPostProcessor接口的Bean对象，调用它们的postProcessAfterInitialization()初始化后方法，如果调用某个对象的初始化方法返回了null，则直接返回

  ```java
  public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
      throws BeansException {
  
      Object result = existingBean;
  
      for (BeanPostProcessor processor : getBeanPostProcessors()) {
          Object current = processor.postProcessAfterInitialization(result, beanName);
          if (current == null) {
              return result;
          }
          result = current;
      }
      return result;
  }
  ```

## 3、实例化

经历了实例化前操作，如果返回的对象不是null，则直接将返回的对象作为Bean实例，如果返回的对象等于null，则调用doCreateBean()方法创建Bean实例

```java
Object beanInstance = doCreateBean(beanName, mbdToUse, args);
```

doCreateBean()中调用createBeanInstance()来完成实例化

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    throws BeanCreationException {
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        // 有可能在本Bean创建之前，就有其他Bean把当前Bean给创建出来了（比如依赖注入过程中）
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 创建Bean实例
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
}
```

具体的实例化过程涉及到根据@Bean注解实例化、以及构造方法推断等，后续构造推断再细讲实例化的过程

## 4、实例化后

在doCreateBean()中调用createBeanInstance()完成Bean的实例化以后，还需要进行实例化后的一系列的操作

- BeanDefinition后置处理

  完成实例化之后，遍历BeanPostProcessorCache中缓存的实现了MergedBeanDefinitionPostProcessor接口的Bean对象，然后调用对象的postProcessMergedBeanDefinition()方法，对合并的BeanDefinition进行操作，包括设置初始化方法名、属性赋值等等操作

  ```java
  protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {
      ……
          synchronized (mbd.postProcessingLock) {
          if (!mbd.postProcessed) {
              applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
              mbd.postProcessed = true;
          }
      } 
      ……
  }
  
  protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
      for (MergedBeanDefinitionPostProcessor processor : getBeanPostProcessorCache().mergedDefinition) {
          processor.postProcessMergedBeanDefinition(mbd, beanType, beanName);
      }
  }
  ```

- 属性填充

  执行完BeanDefinition的后置处理后，将进行属性填充，而属性填充又涉及到一系列的动作

  ```java
  protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {
      ……
      // 属性填充
      populateBean(beanName, mbd, instanceWrapper);
      ……
  }
  ```

  **调用实例化后方法：**

  这个地方同实例化前操作相似，不过这里调用的是实现了InstantiationAwareBeanPostProcessor接口的Bean对象的postProcessAfterInstantiation()实例化后方法

  ```java
  protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
      ……
      if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
          for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
              if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                  return;
              }
          }
      }
  }
  ```

  **属性填充：**

  AutowiredAnnotationBeanPostProcessor和CommonAnnotationBeanPostProcessor都继承了InstantiationAwareBeanPostProcessor接口，而这两个类会在postProcessProperties()方法中解决@Autowired和@Resource的属性注入，代码中的pvs得到的是在BeanDefinition后置处理中设置属性值，如果设置了属性值，AutowiredAnnotationBeanPostProcessor将不在处理这些属性，直接返回

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
              }
              pvs = pvsToUse;
          }
      }
  }
  ```

- 初始化

  属性填充完成之后，进行初始化的操作

  ```java
  protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {
      ……
      // 初始化
  	exposedObject = initializeBean(beanName, exposedObject, mbd);
      ……
  }
  ```

  **调用Aware方法：**

  在进行初始化的时候，首先判断当前Bean实现了哪些Aware接口，然后设置对应的属性

  ```java
  protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
      ……
      invokeAwareMethods(beanName, bean);
      ……
  }
  
  private void invokeAwareMethods(String beanName, Object bean) {
      if (bean instanceof Aware) {
          if (bean instanceof BeanNameAware) {
              ((BeanNameAware) bean).setBeanName(beanName);
          }
          if (bean instanceof BeanClassLoaderAware) {
              ClassLoader bcl = getBeanClassLoader();
              if (bcl != null) {
                  ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
              }
          }
          if (bean instanceof BeanFactoryAware) {
              ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
          }
      }
  }
  ```

  **调用初始化前方法：**

  Aware方法调用完成之后，调用初始化前方法，执行所有BeanPostProcessor的postProcessBeforeInitialization()初始化前方法，如果某个Bean对象调用该方法返回null，则直接返回，不再执行后面Bean对象的方法

  ```java
  protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
      ……
      if (mbd == null || !mbd.isSynthetic()) {
          // 初始化前
          wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
      }
      ……
  }
  public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
      throws BeansException {
  
      Object result = existingBean;
      for (BeanPostProcessor processor : getBeanPostProcessors()) {
          Object current = processor.postProcessBeforeInitialization(result, beanName);
          if (current == null) {
              return result;
          }
          result = current;
      }
      return result;
  }
  ```

  **调用初始化方法：**

  初始化的过程中，会判断当前Bean实例是否实现了InitializingBean接口，如果实现了该接口，则会调用Bean对象的afterPropertiesSet()方法，如果在BeanDifinition的后置处理中设置了初始化方法名并且该对象中有此方法，放回通过反射调用指定的初始化方法

  ```java
  protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
      ……
      // 初始化
      invokeInitMethods(beanName, wrappedBean, mbd);
      ……
  }
  protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd)
      throws Throwable {
  
      boolean isInitializingBean = (bean instanceof InitializingBean);
      if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
          ((InitializingBean) bean).afterPropertiesSet();
      }
  
      if (mbd != null && bean.getClass() != NullBean.class) {
          String initMethodName = mbd.getInitMethodName();
          if (StringUtils.hasLength(initMethodName) &&
              !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
              !mbd.isExternallyManagedInitMethod(initMethodName)) {
              invokeCustomInitMethod(beanName, bean, mbd);
          }
      }
  }
  ```

  **调用初始化后：**

  初始化完成之后，调用初始化后方法，执行所有BeanPostProcessor的postProcessAfterInitialization()初始化后方法，如果某个Bean对象调用该方法返回null，则直接返回，不再执行后面Bean对象的方法

  ```java
  protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
      ……
      // 初始化后
      if (mbd == null || !mbd.isSynthetic()) {
          wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
      }
      ……
  }
  public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
      throws BeansException {
  
      Object result = existingBean;
  
      for (BeanPostProcessor processor : getBeanPostProcessors()) {
          Object current = processor.postProcessAfterInitialization(result, beanName);
          if (current == null) {
              return result;
          }
          result = current;
      }
      return result;
  }
  ```

- 注册Bean销毁逻辑

  在后续Bean销毁会讲到注册Bean销毁逻辑以及Bean的销毁
  

`至此,一个完整的Bean对象创建完成,这一片篇文章主要讲述的是Bean的生命周期,而里面涉及到的更详细的推断构造方法、依赖注入和解决循环依赖这些核心的实现将在后续章节详细介绍`



## 5、总结

创建一个Bean的大概过程/Bean生命周期：

1、调用AbstractBeanFactory.resolveBeanClass()加载类

2、调用InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation() ，实例化前方法

3、调用AbstractAutowireCapableBeanFactory.createBeanInstance() 通过推断构造，完成实例化

4、调用MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()，完成合并的BeanDefinition的后置处理(设置初始化方法名、属性等)

5、调用InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()，执行实例化后的逻辑

6、调用InstantiationAwareBeanPostProcessor.postProcessProperties()，完成属性填充

7、调用AbstractAutowireCapableBeanFactory.invokeAwareMethods()，完成相关属性设置

8、调用BeanPostProcessor.postProcessBeforeInitialization()，完成初始化前操作，这一步包含了很多的属性设置，postProcessBeforeInitialization()的实现类有很多，都是在这一步完成操作

9、调用InitializingBean.afterPropertiesSet()，以及初始化方法

10、调用BeanPostProcessor.postProcessAfterInitialization()初始化后方法

## 6、@PostConstruct和@PreDestroy方法执行

定义了这两个注解的方法都会在执行初始化前操作中完成缓存同时执行@PostConstruct方法，Spring中默认有很多实现了BeanPostProcessor的接口的类，它们在Spring启动的过程中，调用调用postProcessBeforeInitialization()

其中这两个注解方法的缓存以及@PostConstruct注解方法的执行都是在InitDestroyAnnotationBeanPostProcessor类的postProcessBeforeInitialization()中

首先调用findLifecycleMetadata()方法，找到所有当前类以及其父类中定义@PostConstruct和@PreDestroy的方法，然后调用@PostConstruct定义的方法(初始化方法)

```java
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
    //调用初始化方法
    metadata.invokeInitMethods(bean, beanName);
    return bean;
}
```

findLifecycleMetadata()的具体实现如下，lifecycleMetadataCache是@PostConstruct和@PreDestroy的方法的缓存池，其中KEY为beanClass，VALUE为LifecycleMetadata实例对象，而LifecycleMetadata中存放了benaClass对应的@PostConstruct和@PreDestroy的方法，先根据benaClass从缓存中取，如果没有再扫描获取这些方法

```java
private LifecycleMetadata findLifecycleMetadata(Class<?> clazz) {
    if (this.lifecycleMetadataCache == null) {
        // Happens after deserialization, during destruction...
        return buildLifecycleMetadata(clazz);
    }
    // Quick check on the concurrent map first, with minimal locking.
    LifecycleMetadata metadata = this.lifecycleMetadataCache.get(clazz);
    if (metadata == null) {
        synchronized (this.lifecycleMetadataCache) {
            metadata = this.lifecycleMetadataCache.get(clazz);
            if (metadata == null) {
                metadata = buildLifecycleMetadata(clazz);
                this.lifecycleMetadataCache.put(clazz, metadata);
            }
            return metadata;
        }
    }
    return metadata;
}

// LifecycleMetadata 结构如下
private class LifecycleMetadata {

    private final Class<?> targetClass;

    private final Collection<LifecycleElement> initMethods;

    private final Collection<LifecycleElement> destroyMethods;

    @Nullable
    private volatile Set<LifecycleElement> checkedInitMethods;

    @Nullable
    private volatile Set<LifecycleElement> checkedDestroyMethods;
    ……
}
// 具体的方法存放在LifecycleElement实例中
private static class LifecycleElement {

    private final Method method;

    private final String identifier;
    ……
}
```

如果当前lifecycleMetadataCache缓存中找不到对应的方法，则调用buildLifecycleMetadata(clazz)来扫描beanClass中所有的方法以及父类中的方法，判断是否有方法添加了@PostConstruct和@PreDestroy

```java
private LifecycleMetadata buildLifecycleMetadata(final Class<?> clazz) {
    List<LifecycleElement> initMethods = new ArrayList<>();
    List<LifecycleElement> destroyMethods = new ArrayList<>();
    Class<?> targetClass = clazz;

    do {
        final List<LifecycleElement> currInitMethods = new ArrayList<>();
        final List<LifecycleElement> currDestroyMethods = new ArrayList<>();

        ReflectionUtils.doWithLocalMethods(targetClass, method -> {
            if (this.initAnnotationType != null && method.isAnnotationPresent(this.initAnnotationType)) {
                LifecycleElement element = new LifecycleElement(method);
                currInitMethods.add(element);
            }
            if (this.destroyAnnotationType != null && method.isAnnotationPresent(this.destroyAnnotationType)) {
                currDestroyMethods.add(new LifecycleElement(method));
            }
        });

        // 父类的在前面,执行初始化方法时,先执行父类的初始化方法
        initMethods.addAll(0, currInitMethods);
        destroyMethods.addAll(currDestroyMethods);
        // 先遍历当前类的方法,再遍历父类的方法
        targetClass = targetClass.getSuperclass();
    }
    while (targetClass != null && targetClass != Object.class);

    return (initMethods.isEmpty() && destroyMethods.isEmpty() ? this.emptyLifecycleMetadata :
            new LifecycleMetadata(clazz, initMethods, destroyMethods));
}
```



