## 一、实例化所有非懒加载的单例

不论是基于注解的Spring容器，还是基于xml的Spring容器，在启动的过程中，都会调用AbstractApplicationContext的refresh()，在该方法中，通过调用finishBeanFactoryInitialization(beanFactory)来实例化所有非懒加载的单例Bean

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    ……
    // Instantiate all remaining (non-lazy-init) singletons.
    // 实例化非懒加载的单例Bean
    beanFactory.preInstantiateSingletons();
}
```

#### 1.1 实例化普通的单例Bean

Spring扫描注册类的BeanDefinition的时候，会将beanName也保存一份，存在beanDefinitionNames中，这是一个ArrayList，实例化非懒加载的单例Bean时，遍历beanNam的列表，获取对应的BeanDefinition

```java
public void preInstantiateSingletons() throws BeansException {

    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    // Trigger initialization of all non-lazy singleton beans...
    for (String beanName : beanNames) {
        // 获取合并后的BeanDefinition
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);

        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                ……
            }
            else {
                // 创建Bean对象
                getBean(beanName);
            }
        }
    }
```



- 根据beanName获取合并后的BeanDefinition，什么是合并后的BeanDefinition呢？

  在Spring.xml中定义两个Bean，其中userService继承自user，因此如果userService不显示指定scope属性，那么它将继承user的scope属性，变成一个原型Bean。

  ```java
  <bean id="user" class="com.lizhi.service.User" scope="prototype"/>
  
  <bean id="userService" class="com.lizhi.service.UserService" parent="user"/>
  ```

  Spring会为下面两个Bean都创建各自的BeanDefinition，但是再实例化的时候，如果为某个Bean指定了parent，则需要整合它所有上层Bean的属性，生成一个RootBeanDefinition，如果没有指定parent，也需要根据当前Bean的BeanDefinition生成一个RootBeanDefinition，然后放入到mergedBeanDefinitions中，正真实例化用的BeanDefinition其实是合并后的RootBeanDefinition

  

  getMergedLocalBeanDefinition(beanName)源码如下：

  先从mergedBeanDefinitions中根据beanName获取RootBeanDefinition，如果没有再生成

  ```java
  protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
      // Quick check on the concurrent map first, with minimal locking.
      RootBeanDefinition mbd = this.mergedBeanDefinitions.get(beanName);
      if (mbd != null && !mbd.stale) {
          return mbd;
      }
      return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
  }
  ```

  获取合并后的RootBeanDefinition，getParentName()判断是否指定parant属性，如果没有，则直接根据当前BeanDefinition生成RootBeanDefinition；如果指定了pareant属性，则通过pbd = getMergedBeanDefinition(parentBeanName);递归获取上级父BeanDefinition，然后根据父BeanDefinition生成一个新的RootBeanDefinition，再用当前BeanDefinition的属性去覆盖新生成的RootBeanDefinition，然后将新生成的RootBeanDefinition放入到mergedBeanDefinitions中

  ```java
  protected RootBeanDefinition getMergedBeanDefinition(
      String beanName, BeanDefinition bd, @Nullable BeanDefinition containingBd)
      throws BeanDefinitionStoreException {
  
      synchronized (this.mergedBeanDefinitions) {
          RootBeanDefinition mbd = null;
          RootBeanDefinition previous = null;
  
          // Check with full lock now in order to enforce the same merged instance.
          if (containingBd == null) {
              mbd = this.mergedBeanDefinitions.get(beanName);
          }
  
          if (mbd == null || mbd.stale) {
              previous = mbd;
              if (bd.getParentName() == null) {
                  // Use copy of given root bean definition.
                  if (bd instanceof RootBeanDefinition) {
                      mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();
                  }
                  else {
                      mbd = new RootBeanDefinition(bd);
                  }
              }
              else {
                  // Child bean definition: needs to be merged with parent.
                  // pbd表示parentBeanDefinition
                  BeanDefinition pbd;
                  try {
                      String parentBeanName = transformedBeanName(bd.getParentName());
                      if (!beanName.equals(parentBeanName)) {
                          // 递归获取父BeanDefinition,当前BeanDefinition的父BeanDefinition可能
                          pbd = getMergedBeanDefinition(parentBeanName);
                      }
                      else {
                          BeanFactory parent = getParentBeanFactory();
                          if (parent instanceof ConfigurableBeanFactory) {
                              pbd = ((ConfigurableBeanFactory) parent).getMergedBeanDefinition(parentBeanName);
                          }
                          ……
                      }
                      ……
                  }
  
                  // Deep copy with overridden values.
                  // 子BeanDefinition的属性覆盖父BeanDefinition的属性，这就是合并
                  mbd = new RootBeanDefinition(pbd);
                  mbd.overrideFrom(bd);
              }
              ……
              //将新生成的RootBeanDefinition放入到mergedBeanDefinitions
              if (containingBd == null && isCacheBeanMetadata()) {
                  this.mergedBeanDefinitions.put(beanName, mbd);
              }
          }
          ……
  		return mbd;
      }
  ```

- 判断是否应该实例化

  获取RootBeanDefinition，判断是否是单例，是否是非懒加载，isAbstract()则用与判断当前BeanDefinition是否为抽象的BeanDefinition

  ```java
  RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
  
  if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
      if (isFactoryBean(beanName)) {
          // 获取FactoryBean对象
          Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
          if (bean instanceof FactoryBean) {
              // FactoryBean实例化
              ……
          }
      }
      else {
          // 创建普通Bean对象
          getBean(beanName);
      }
  }
  ```

  通过Spring.xml方式配置Bean，可以通过**abstract**属性指定抽象的Beandefinition

  ```xml
  <bean id="user" class="com.lizhi.service.User" scope="prototype" abstract="true"/>
  ```

  符合上述条件，则调用getBean(beanName)来完成非懒加载单例Bean的实例化

#### 1.2 实例化FactoryBean

获取RootBeanDefinition，首先需要判断判断是否是一个FactoryBean，开发人员可以通过实现FactoryBean接口自定义Bean的创建

```java
// 获取合并后的BeanDefinition
RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);

if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
    if (isFactoryBean(beanName)) {
        // 获取FactoryBean对象
        Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
        if (bean instanceof FactoryBean) {
            FactoryBean<?> factory = (FactoryBean<?>) bean;
            boolean isEagerInit;
            ……
            isEagerInit = (factory instanceof SmartFactoryBean &&
                               ((SmartFactoryBean<?>) factory).isEagerInit());
            if (isEagerInit) {
                // 创建真正的Bean对象(getObject()返回的对象)
                getBean(beanName);
            }
        }
        // 实例化普通Bean
        ……
    }
}
```

- 根据beanName判断是否是FactoryBean

  首先去从单例池中获取Bean，判断Bean是否实现了FactoryBean接口，因为现在处于实例化单例Bean的过程，所以当前单例池是没有的

  单例池如果没有的话，则判断当前BeanDefinitionMap中是否有该BeanName，如果当前BeanFactory的BeanDefinitionMap中没有，则去ParentBeanFactory的BeanDefinitionMap中查找

  `什么是ParentBeanFactory?`

  如下代码所示，AppConfig1和AppConfig配置类指定扫描不同的包，生成了两个Spring容器，context容器将parent设置为自己的父容器，当从context容器中找不到资源时就会去parent找

  ```java
  AnnotationConfigApplicationContext parent = new AnnotationConfigApplicationContext(AppConfig1.class);
  
  AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
  context.setParent(parent);
  ```

  父容器的单例池中也没有Bean，就需要根据beanName和RootBeanDefinition去推断Bean类型

  ```java
  public boolean isFactoryBean(String name) throws NoSuchBeanDefinitionException {
      String beanName = transformedBeanName(name);
      Object beanInstance = getSingleton(beanName, false);
      if (beanInstance != null) {
          return (beanInstance instanceof FactoryBean);
      }
      // No singleton instance found -> check bean definition.
      if (!containsBeanDefinition(beanName) && getParentBeanFactory() instanceof ConfigurableBeanFactory) {
          // No bean definition found in this factory -> delegate to parent.
          return ((ConfigurableBeanFactory) getParentBeanFactory()).isFactoryBean(name);
      }
      return isFactoryBean(beanName, getMergedLocalBeanDefinition(beanName));
  }
  ```

  推断beanName是否是FactoryBean，然后将推断结果进行缓存，下次就可以直接使用了，BeanDefinition中的很多属性都是作为缓存使用的

  ```java
  protected boolean isFactoryBean(String beanName, RootBeanDefinition mbd) {
      Boolean result = mbd.isFactoryBean;
      if (result == null) {
          // 根据BeanDefinition推测Bean类型（获取BeanDefinition的beanClass属性）
          Class<?> beanType = predictBeanType(beanName, mbd, FactoryBean.class);
          // 判断是不是实现了FactoryBean接口
          result = (beanType != null && FactoryBean.class.isAssignableFrom(beanType));
          // 缓存当前Bean是否为FactoryBean
          mbd.isFactoryBean = result;
      }
      return result;
  }
  ```

- 获取FactoryBean对象

  `如果getBean()中的beanName不是以&开头,则返回的是FactoryBean的getObject()返回的对象;如果beanName是以&开头的,则返回的是FactoryBean对象`

  想要获取FactoryBean对象，需要在beanName前加上“&”，表示这是FactoryBean的beanName

  ```java
  if (isFactoryBean(beanName)) {
      // 获取FactoryBean对象
      Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
      ……
  }
  ```

  根据转换后的beanName从单例池中找Bean，转化后的beanName是没有"&"前缀的，如果能从单例池中取出单例Bean，再调用getObjectForBeanInstance()

  ```java
  protected <T> T doGetBean(
      String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
      throws BeansException {
  
      // name有可能是 &xxx 或者 xxx，如果name是&xxx，那么beanName就是xxx
      // name有可能传入进来的是别名，那么beanName就是id
      String beanName = transformedBeanName(name);
      Object beanInstance;
      Object sharedInstance = getSingleton(beanName);
      if (sharedInstance != null && args == null) {
          // 如果sharedInstance是FactoryBean，那么就调用getObject()返回对象
          beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
      }
      // 创建FactoryBean或普通Bean
      ……
  ```

  如果需要获取FactoryBean对象，同时传进来的beanInstance就是FactoryBean则直接返回

  ```java
  protected Object getObjectForBeanInstance(
      Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
  
      // Don't let calling code try to dereference the factory if the bean isn't a factory.
      // 如果&xxx，那么就直接返回单例池中的对象
      if (BeanFactoryUtils.isFactoryDereference(name)) {
          if (beanInstance instanceof NullBean) {
              return beanInstance;
          }
          if (!(beanInstance instanceof FactoryBean)) {
              throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
          }
          if (mbd != null) {
              mbd.isFactoryBean = true;
          }
          return beanInstance;
      }
      ……
  ```

  

  transformedBeanName()就是将传进来带&的beanName，去掉前面的&作为beanName，单例池中的Key是对应的beanName是xxx格式

  ```java
  // FACTORY_BEAN_PREFIX = "&"
  public static String transformedBeanName(String name) {
      Assert.notNull(name, "'name' must not be null");
      // 如果不是FactoryBean格式的beanName就直接返回
      if (!name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
          return name;
      }
      return transformedBeanNameCache.computeIfAbsent(name, beanName -> {
          do {
              // beanName前面可能有多个&前缀,去掉所有的&前缀
              beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
          }
          while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));
          return beanName;
      });
  }
  ```

- 判断获取的Bean对象是否为FactoryBean

  如果获取到是FactoryBean对象，接着判断该FactoryBean对象是否实现了SmartFactoryBean接口，如果实现了该接口，并且接口中isEagerInit()返回的是true，意味着Spring容器启动的过程中，就会调用FactoryBean对象的getObject()创建对应的Bean对象，如果没有实现SmartFactoryBean接口或isEagerInit()返回的是false，则在调用getBean()时才会调用getObject()创建对应的Bean对象

  ```java
  if (bean instanceof FactoryBean) {
      FactoryBean<?> factory = (FactoryBean<?>) bean;
      boolean isEagerInit;
      ……
      isEagerInit = (factory instanceof SmartFactoryBean &&
                     ((SmartFactoryBean<?>) factory).isEagerInit());
      if (isEagerInit) {
          // 创建真正的Bean对象(getObject()返回的对象)
          getBean(beanName);
      }
  }
  ```

  `FactoryBean的实现类,在Spring加载的时候,只会生成该实现类类的BeanDifiniton,而getObject()中的类是没有BeanDifiniton的,并且这种方式创建的Bean没有完整的生命周期,只是经历了初始化后`

#### 1.3 实例化后方法调用

当所有非懒加载的单例Bean创建完成之后，遍历所有的单例Bean，判断是否实现了SmartInitializingSingleton接口，如果某个单例Bean实现了该接口，则会去调用该接口的afterSingletonsInstantiated()

```java
public void preInstantiateSingletons() throws BeansException {
    // 所有的非懒加载单例Bean都创建完了后
    // Trigger post-initialization callback for all applicable beans...
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            // JFR(Java Flight Record) 技术,用于监控记录以下代码的执行
            StartupStep smartInitialize = this.getApplicationStartup().start("spring.beans.smart-initialize")
                .tag("beanName", beanName);
            SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                    smartSingleton.afterSingletonsInstantiated();
                    return null;
                }, getAccessControlContext());
            }
            else {
                smartSingleton.afterSingletonsInstantiated();
            }
            smartInitialize.end();
        }
    }
}
```



