## 一、注册Bean销毁逻辑

Bean的生命周期中，在完成了Bean的创建之后，会注册Bean销毁的逻辑

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    throws BeanCreationException {
    ……
    registerDisposableBeanIfNecessary(beanName, bean, mbd);
    ……
}
```

在注册Bean销毁逻辑时，首先需要通过!mbd.isPrototype()保证该Bean不是原型Bean，因为Spring容器并不会缓存原型Bean，就没有销毁一说

接着通过requiresDestruction()判断该Bean是否需要销毁，对需要销毁的Bean通过适配器模式生成DisposableBeanAdapter对象，最后调用registerDisposableBean()将DisposableBeanAdapter对象放入disposableBeans缓存中，当Spring容器关闭的时候，可以直接从该缓存中取出销毁调用，调用它们的销毁方法

```java
	protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
		AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
		if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
			if (mbd.isSingleton()) {
				// Register a DisposableBean implementation that performs all destruction
				// work for the given bean: DestructionAwareBeanPostProcessors,
				// DisposableBean interface, custom destroy method.
				registerDisposableBean(beanName, new DisposableBeanAdapter(
						bean, beanName, mbd, getBeanPostProcessorCache().destructionAware, acc));
			}
            ……
		}
	}
```

#### 1.1 判断当前Bean是否需要销毁

requiresDestruction()方法源码如下：

```java
protected boolean requiresDestruction(Object bean, RootBeanDefinition mbd) {
    return (bean.getClass() != NullBean.class && (DisposableBeanAdapter.hasDestroyMethod(bean, mbd) ||
                                                  (hasDestructionAwareBeanPostProcessors() && DisposableBeanAdapter.hasApplicableProcessors(
                                                      bean, getBeanPostProcessorCache().destructionAware))));
}
```

- 判断当前Bean是否有销毁方法

  如果当前Bean继承了DisposableBean或AutoCloseable接口，重写接口中的destroy()和close()方法，这两个方法都是销毁方法

  ```java
  public static boolean hasDestroyMethod(Object bean, RootBeanDefinition beanDefinition) {
      if (bean instanceof DisposableBean || bean instanceof AutoCloseable) {
          return true;
      }
      return inferDestroyMethodIfNecessary(bean, beanDefinition) != null;
  }
  ```

  如果没有继承这两个接口，则判断当前Bean的RootBeanDefinition是否设置销毁方法名为"(inferred)"

  设置初始化和销毁方法名都是在Bean实例化后BeanDefinition的后置处理，即MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()中完成的

  如果在BeanDefinition的后置处理中设置销毁方法名为"(inferred)"，则会将该Bean中close()和shutdown()作为销毁方法(前提是Bean里面有这两个方法)

  ```java
  private static String inferDestroyMethodIfNecessary(Object bean, RootBeanDefinition beanDefinition) {
      String destroyMethodName = beanDefinition.resolvedDestroyMethodName;
      if (destroyMethodName == null) {
          destroyMethodName = beanDefinition.getDestroyMethodName(); //
          if (AbstractBeanDefinition.INFER_METHOD.equals(destroyMethodName)) {
              destroyMethodName = null;
              if (!(bean instanceof DisposableBean)) {
                  try {
                      destroyMethodName = bean.getClass().getMethod(CLOSE_METHOD_NAME).getName();
                  }
                  catch (NoSuchMethodException ex) {
                      destroyMethodName = bean.getClass().getMethod(SHUTDOWN_METHOD_NAME).getName();
                  }
              }
          }
          beanDefinition.resolvedDestroyMethodName = (destroyMethodName != null ? destroyMethodName : "");
      }
      return (StringUtils.hasLength(destroyMethodName) ? destroyMethodName : null);
  }
  ```

- 判断是否有Bean实现了DestructionAwareBeanPostProcessor接口且requiresDestruction()方法返回True

  DestructionAwareBeanPostProcessor接口主要用于Bean销毁的，其中的requiresDestruction()判断Bean是否需要销毁，而postProcessBeforeDestruction()实现具体的销毁逻辑

  在[《创建Bean》](https://blog.csdn.net/sermonlizhi/article/details/120748334)的最后面，讲到了@PreDestroy注解，它主要用于定义销毁方法(被该注解修饰的方法都是销毁方法)，而该注解的扫描的是在InitDestroyAnnotationBeanPostProcessor类中完成，InitDestroyAnnotationBeanPostProcessor实现了DestructionAwareBeanPostProcessor接口，它会缓存每个Bean以及它的父类哪些方法是被@PreDestroy修饰的

  hasDestructionAwareBeanPostProcessors()主要判断是否缓存的有实现了DestructionAwareBeanPostProcessor的Bean

  ```java
  protected boolean hasDestructionAwareBeanPostProcessors() {
      return !getBeanPostProcessorCache().destructionAware.isEmpty();
  }
  ```

  然后调用DestructionAwareBeanPostProcessor的requiresDestruction()判断是否需要销毁

  ```java
  public static boolean hasApplicableProcessors(Object bean, List<DestructionAwareBeanPostProcessor> postProcessors) {
      if (!CollectionUtils.isEmpty(postProcessors)) {
          for (DestructionAwareBeanPostProcessor processor : postProcessors) {
              if (processor.requiresDestruction(bean)) {
                  return true;
              }
          }
      }
      return false;
  }
  ```

  以InitDestroyAnnotationBeanPostProcessor.requiresDestruction(bean)为例，查看其实现过程：

  在[《创建Bean》](https://blog.csdn.net/sermonlizhi/article/details/120748334)中介绍@PostConstruct和@PreDestroy的时候，已经详细介绍过findLifecycleMetadata()，通过判断是否有@PreDestroy定义的销毁方法，判断当前Bean是否需要销毁

  ```java
  public boolean requiresDestruction(Object bean) {
      return findLifecycleMetadata(bean.getClass()).hasDestroyMethods();
  }
  
  private LifecycleMetadata findLifecycleMetadata(Class<?> clazz) {
      if (this.lifecycleMetadataCache == null) {
          // Happens after deserialization, during destruction...
          return buildLifecycleMetadata(clazz);
      }
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
  ```

#### 1.2 注册可以销毁的Bean

判断完Bean是否可以销毁之后，需要注册销毁的Bean，代码如下：

registerDisposableBean()中缓存的是DisposableBeanAdapter对象，即不论该Bean是实现了DisposableBean或AutoCloseable接口，或者是通过BeanDifinition后置处理指定了”(inferred)“销毁方法名或其他方法销毁方法， 还是通过@PreDestroy指定的销毁方法，对于各种销毁逻辑，这里都会将Bean适配成一个DisposableBeanAdapter对象

```java
// Register a DisposableBean implementation that performs all destruction
// work for the given bean: DestructionAwareBeanPostProcessors,
// DisposableBean interface, custom destroy method.
registerDisposableBean(beanName, new DisposableBeanAdapter(
    bean, beanName, mbd, getBeanPostProcessorCache().destructionAware, acc));
```

```java
public void registerDisposableBean(String beanName, DisposableBean bean) {
    synchronized (this.disposableBeans) {
        this.disposableBeans.put(beanName, bean);
    }
}
```



在DisposableBeanAdapter的构造方法中，会推断出构造方法，并过滤出所有实现了DestructionAwareBeanPostProcessor接口但requiresDestruction()方法返回True的Bean，销毁的时候会调用它们的postProcessBeforeDestruction()

```java
public DisposableBeanAdapter(Object bean, String beanName, RootBeanDefinition beanDefinition,
                             List<DestructionAwareBeanPostProcessor> postProcessors, @Nullable AccessControlContext acc) {
    ……
    String destroyMethodName = inferDestroyMethodIfNecessary(bean, beanDefinition);
    if (destroyMethodName != null && !(this.invokeDisposableBean && "destroy".equals(destroyMethodName)) &&
        !beanDefinition.isExternallyManagedDestroyMethod(destroyMethodName)) {
        this.destroyMethodName = destroyMethodName;
        Method destroyMethod = determineDestroyMethod(destroyMethodName);
        if (destroyMethod == null) {
            ……
        }
        else {
            ……
            destroyMethod = ClassUtils.getInterfaceMethodIfPossible(destroyMethod);
        }
        this.destroyMethod = destroyMethod;
    }
    this.beanPostProcessors = filterPostProcessors(postProcessors, bean);
}
```

```java
private static List<DestructionAwareBeanPostProcessor> filterPostProcessors(
    List<DestructionAwareBeanPostProcessor> processors, Object bean) {

    List<DestructionAwareBeanPostProcessor> filteredPostProcessors = null;
    if (!CollectionUtils.isEmpty(processors)) {
        filteredPostProcessors = new ArrayList<>(processors.size());
        for (DestructionAwareBeanPostProcessor processor : processors) {
            if (processor.requiresDestruction(bean)) {
                filteredPostProcessors.add(processor);
            }
        }
    }
    return filteredPostProcessors;
}
```

在Bean销毁的时候，会调用DisposableBeanAdapter的destroy()，这个销毁方法里面会执行各种销毁逻辑

```java
public void destroy() {
    // 调用所有DestructionAwareBeanPostProcessor.postProcessBeforeDestruction方法
    if (!CollectionUtils.isEmpty(this.beanPostProcessors)) {
        for (DestructionAwareBeanPostProcessor processor : this.beanPostProcessors) {
            // @PreDestroy定义的销毁方法就是在这一步执行
            processor.postProcessBeforeDestruction(this.bean, this.beanName);
        }
    }
    ……
}
```



## 二、Bean销毁过程

在Spring容器关闭的时候，会去销毁所有的单例Bean，并不是只有注册了销毁逻辑的Bean才被销毁，注册了销毁逻辑的单例Bean在销毁之前，会调用它们注册的销毁逻辑

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
context.close();
```

容器的close()会调用doClose()，doClose()会调用来销毁单例Bean

```java
public void close() {
    synchronized (this.startupShutdownMonitor) {
        doClose();
        ……
    }
}

protected void doClose() {
    ……
    destroyBeans();
    ……
}
```

在destroySingletons()中会取出disposableBeans缓存中定了销毁逻辑的Bean的beanName，然后遍历进行销毁

```java
protected void destroyBeans() {
    getBeanFactory().destroySingletons();
}

public void destroySingletons() {
    ……
    String[] disposableBeanNames;
    synchronized (this.disposableBeans) {
        disposableBeanNames = StringUtils.toStringArray(this.disposableBeans.keySet());
    }
    for (int i = disposableBeanNames.length - 1; i >= 0; i--) {
        destroySingleton(disposableBeanNames[i]);
    }
    ……
}
```

在进行销毁的时候，先从单例池等缓存中移除Bean，然后从disposableBeans移除当前DisposableBean并获取该对象，然后调用destroyBean(beanName, disposableBean)执行对象注册的销毁逻辑

```java
public void destroySingleton(String beanName) {
    // Remove a registered singleton of the given name, if any.
    // 先从单例池中移除掉
    removeSingleton(beanName);

    // Destroy the corresponding DisposableBean instance.
    DisposableBean disposableBean;
    synchronized (this.disposableBeans) {
        disposableBean = (DisposableBean) this.disposableBeans.remove(beanName);
    }
    destroyBean(beanName, disposableBean);
}
// 从缓存中移除Bean
protected void removeSingleton(String beanName) {
    synchronized (this.singletonObjects) {
        this.singletonObjects.remove(beanName);
        this.singletonFactories.remove(beanName);
        this.earlySingletonObjects.remove(beanName);
        this.registeredSingletons.remove(beanName);
    }
}
```

在销毁当前Bean的时候，会获取依赖当前Bean的其他Bean的beanName，然后递归调用destroySingleton()方法，保证没有被任何其他Bean依赖的Bean先销毁，在进行销毁时，会先调用DisposableBean的destroy()方法，注册Bean销毁逻辑时已经讲过该方法，然后再去调用其他的销毁逻辑，其他的销毁逻辑无非就是从各种缓存中根据BeanName，清除缓存

```java
protected void destroyBean(String beanName, @Nullable DisposableBean bean) {

    // dependentBeanMap表示某bean被哪些bean依赖了
    // 所以现在要销毁某个bean时，如果这个Bean还被其他Bean依赖了，那么也得销毁其他Bean
    // Trigger destruction of dependent beans first...
    Set<String> dependencies;
    synchronized (this.dependentBeanMap) {
        // Within full synchronization in order to guarantee a disconnected Set
        dependencies = this.dependentBeanMap.remove(beanName);
    }
    if (dependencies != null) {
        if (logger.isTraceEnabled()) {
            logger.trace("Retrieved dependent beans for bean '" + beanName + "': " + dependencies);
        }
        for (String dependentBeanName : dependencies) {
            destroySingleton(dependentBeanName);
        }
    }

    // Actually destroy the bean now...
    if (bean != null) {
        bean.destroy();
    }
    ……
}
```

