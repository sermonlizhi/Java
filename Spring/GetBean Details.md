Spring容器提供了五种获取Bean的方式，可以根据beanName获取Bean，也可以根据classType来获取Bean，所有根据beanName来获取Bean的方式，底层都会通过调用下面的doGetBean()来获取Bean对象

```java
protected <T> T doGetBean(
      String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
```

## 一、根据beanName获取Bean

Spring容器提供了三种根据beanName获取Bean的方式：

```java
// 启动Spring容器
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
// 直接根据beanName来获取
UserService userService = (UserService) context.getBean("userService");
// 根据beanName获取Bean后,判断该Bean是否属于UserService类型
UserService userService1 = (UserService) context.getBean("userService",UserService.class);
// 指定构造方法参数,使用指定的构造方法的生成Bean,如果UserService是单例的,则容器启动的过程中就会去自行推断构造方法然后创建单例Bean;如果UserService是原型的，则每次都会getBean()时,会根据后面提供的构造方法参数使用指定的构造方法创建Bean
UserService userService2 = (UserService) context.getBean("userService",new OrderService());
```

上面三种方式都会调用doGetBean()来获取Bean对象，下面将详细介绍doGetBean()的实现

```java
protected <T> T doGetBean(
    String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
    throws BeansException {

    // name有可能是 &xxx 或者 xxx，如果name是&xxx，那么beanName就是xxx
    // name有可能传入进来的是别名，那么beanName就是id
    String beanName = transformedBeanName(name);
    Object beanInstance;

    // Eagerly check singleton cache for manually registered singletons.
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        // 如果sharedInstance是FactoryBean，那么就调用getObject()返回对象
        beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }
```

上述代码在[《实例化单例Bean》](https://blog.csdn.net/sermonlizhi/article/details/120746967)的FactoryBean创建部分有大概介绍过，下面将详细介绍核心的代码方法

#### 1.1 name转换，获取真正的beanName

transformedBeanName()主要将传进来的name转换成真正的beanName，具体实现如下：

```java
protected String transformedBeanName(String name) {
    return canonicalName(BeanFactoryUtils.transformedBeanName(name));
}
```

先看里面的BeanFactoryUtils.transformedBeanName()实现，下面代码在FactoryBean的创建中已经讲过，想要获取一个FactoryBean的对象，那么传进来的name是以“&”前缀开头，可以只有一个“&”，也可以有多个“&”，下面的代码只是将前缀的“&”去掉而已

```java
public static String transformedBeanName(String name) {
    Assert.notNull(name, "'name' must not be null");
    if (!name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
        return name;
    }
    return transformedBeanNameCache.computeIfAbsent(name, beanName -> {
        do {
            beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
        }
        while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));
        return beanName;
    });
}
```

接下来看canonicalName()实现了怎样的功能，调用BeanFactoryUtils.transformedBeanName()返回的可能还不是最终的beanName，可能是Bean的一个别名，使用@Bean注解创建Bean的时候，可以为Bean指定多个名称，如下所示：

Spring会把第一个名字即“userService”作为真正的beanName，而后面的名字作为Bean的别名进行存储

```java
@Bean({"userService","userService1"})
public UserService userService(){
    return new UserService();
}

AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
//根据别名也能获取Bean
UserService userService = (UserService) context.getBean("userService1");
```

Bean的别名存放在一个aliasMap中，其中KEY=别名，VALUE=beanName/别名，根据别名从aliasMap中拿到的可能是真正的beanName，也可能还是一个别名，所以用do-while循环，直到拿出来的名字从aliasMap再找不到对应的值，那么该名字就是真正的beanName了

```java
public String canonicalName(String name) {
    String canonicalName = name;
    // Handle aliasing...
    String resolvedName;
    do {
        resolvedName = this.aliasMap.get(canonicalName);
        if (resolvedName != null) {
            canonicalName = resolvedName;
        }
    }
    while (resolvedName != null);
    return canonicalName;
}
```

#### 1.2 父BeanFactory创建Bean对象

获取到真正的beanName之后，不论是单例Bean还是原型Bean，都会调用getSingleton(beanName)去单例池中查找是否有单例Bean。如果从单例池中取出对象了，接下来的getObjectForBeanInstance()主要根据原始name判断当前Bean是否是FactoryBean，以及获取FactoryBean对象。如果单例池中不存在，则需要创建Bean实例

如果当前BeanFactory的beanDefinitionMap中不存在该beanName，且存在parentBeanFactory，则让parentBeanFactory去创建Bean

```java
BeanFactory parentBeanFactory = getParentBeanFactory();
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
    // Not found -> check parent.
    // &&&&xxx---->&xxx
    String nameToLookup = originalBeanName(name);
    if (parentBeanFactory instanceof AbstractBeanFactory) {
        return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
            nameToLookup, requiredType, args, typeCheckOnly);
    }
    else if (args != null) {
        // Delegation to parent with explicit args.
        return (T) parentBeanFactory.getBean(nameToLookup, args);
    }
    else if (requiredType != null) {
        // No args -> delegate to standard getBean method.
        return parentBeanFactory.getBean(nameToLookup, requiredType);
    }
    else {
        return (T) parentBeanFactory.getBean(nameToLookup);
    }
}
```

originalBeanName(name)方法用于将别名和"&&&"格式的名字进行转换真正的beanName或FactoryBean的beanName

```java
protected String originalBeanName(String name) {
    String beanName = transformedBeanName(name);
    if (name.startsWith(FACTORY_BEAN_PREFIX)) {
        beanName = FACTORY_BEAN_PREFIX + beanName;
    }
    return beanName;
}
```

#### 1.3 当前BeanFactory来创建Bean

从mergedBeanDefinitions中根据beanName获取RootBeanDefinition，[《实例化单例Bean》](https://blog.csdn.net/sermonlizhi/article/details/120746967)中提到过真正创建Bean的时候，用的是合并后的RootBeanDefinition而不是普通的BeanDefinition

然后判断当前RootBeanDefinition是否是抽象的，如果是抽象的将会抛出异常

接着获取该Bean的@DependsOn注解依赖的beanName

- 创建@DependsOn注解依赖Bean对象

  ```java
  RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
  // 检查BeanDefinition是不是Abstract的
  checkMergedBeanDefinition(mbd, beanName, args);
  // Guarantee initialization of beans that the current bean depends on.
  String[] dependsOn = mbd.getDependsOn();
  ```

  DependsOn注解的使用如下：

  DependsOn注解指明在创建UserServicce的Bean之前，需要先创建orderService和user，在Spring生成BeanDefinition的时候，就回去解析@DependsOn，然后将依赖的beanName存放在BeanDefinition的字符串数组dependsOn中

  ```java
  @Component
  @DependsOn({"orderService","user"})
  public class UserService {
  }
  ```

  如果DependsOn数组不为空，将遍历数组，首先判断是否存在@DependsOn的循环依赖(这个地方的循环依赖不同于依赖注入的循环依赖)，即UserService依赖于user，而User的@DependsOn又依赖于userService，对于这种情况，将直接抛出异常

  如果不存在循环依赖，则把这种依赖关系存放在dependentBeanMap中，它的key表示被依赖Bean的beanName，而value是一个Set<String>，存放依赖该Bean的beanName集合，然后调用getBean(dep)来创建依赖的Bean对象

  ```java
  if (dependsOn != null) {
      // dependsOn表示当前beanName所依赖的，当前Bean创建之前dependsOn所依赖的Bean必须已经创建好了
      for (String dep : dependsOn) {
          // beanName是不是被dep依赖了，如果是则出现了循环依赖
          if (isDependent(beanName, dep)) {
              throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                              "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
          }
          // dep被beanName依赖了，存入dependentBeanMap中，dep为key，beanName为value
          registerDependentBean(dep, beanName);
  
          // 创建所依赖的bean
          try {
              getBean(dep);
          }
          catch (NoSuchBeanDefinitionException ex) {
              throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                              "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
          }
      }
  }
  ```

- 创建单例Bean

  如果RootBeanDefinition的scope属性是单例的，则调用getSingleton创建单例Bean

  ```java
  if (mbd.isSingleton()) {
      sharedInstance = getSingleton(beanName, () -> {
          try {
              return createBean(beanName, mbd, args);
          }
          catch (BeansException ex) {
              destroySingleton(beanName);
              throw ex;
          }
      });
      // 前面讲过,主要用于获取FactoryBean
      beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
  }
  ```

  getSingleton(beanName, ObjectFactory<?> singletonFactory)先从单例池中根据beanName来获取单例Bean，如果单例池中没有，则再通过createBean来创建单例Bean

  ```java
  public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
      synchronized (this.singletonObjects) {
          Object singletonObject = this.singletonObjects.get(beanName);
          if (singletonObject == null) {
              ……
              try {
                  // 通过lambda表达式调用createBean方法
                  singletonObject = singletonFactory.getObject();
                  newSingleton = true;
                      }
              ……
              // 添加到单例池
                  if (newSingleton) {
                      addSingleton(beanName, singletonObject);
                  }
          }
          return singletonObject;
      }
  }
  ```

- 创建原型Bean

  如果RootBeanDefinition的scope属性是原型的，则直接调用createBean方法创建原型Bean

  ```java
  if (mbd.isPrototype()) {
      // It's a prototype -> create a new instance.
      Object prototypeInstance = null;
      try {
          beforePrototypeCreation(beanName);
          prototypeInstance = createBean(beanName, mbd, args);
      }
      finally {
          afterPrototypeCreation(beanName);
      }
      beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
  }
  ```

- 创建其他作用域Bean

  除了单例和原型Bean，Spring还支持request、session和application作用域的Bean

  首先获取scopeName，然后根据scopeName获取对应的Scope对象，比如：RequestScope和SessionScope

  Scope对象的get方法都会去调用AbstractRequestAttributesScope类的get()，RequestScope和SessionScope都继承该类

  ```java
  String scopeName = mbd.getScope();
  Scope scope = this.scopes.get(scopeName);
  if (scope == null) {
      throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
  }
  try {  // session.getAttriute(beaName)  setAttri
      Object scopedInstance = scope.get(beanName, () -> {
          beforePrototypeCreation(beanName);
          try {
              return createBean(beanName, mbd, args);
          }
          finally {
              afterPrototypeCreation(beanName);
          }
      });
      beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
  }
  ```

  getScope()返回Scope对象对应的Scope值，然后调用getAttribute方法，如果request和session中都没有Bean实例，

  ```java
  public Object get(String name, ObjectFactory<?> objectFactory) {
      RequestAttributes attributes = RequestContextHolder.currentRequestAttributes();
      Object scopedObject = attributes.getAttribute(name, getScope());
      if (scopedObject == null) {
          // lambda表达式调用createBean创建实例
          scopedObject = objectFactory.getObject();
          // 并将Bean设置到对应的属性中
          attributes.setAttribute(name, scopedObject, getScope());
          ……
      }
      return scopedObject;
  }
  ```

  getAttribute根据传进来的Scope值，判断从request还是session去根据name获取对应属性的对象

  ```java
  public Object getAttribute(String name, int scope) {
      if (scope == SCOPE_REQUEST) {
          if (!isRequestActive()) {
              throw new IllegalStateException(
                  "Cannot ask for request attribute - request is not active anymore!");
          }
          return this.request.getAttribute(name);
      }
      else {
          HttpSession session = getSession(false);
          if (session != null) {
              Object value = session.getAttribute(name);
              if (value != null) {
                  this.sessionAttributesToUpdate.put(name, value);
              }
              return value;
          }
          return null;
      }
  }
  ```

