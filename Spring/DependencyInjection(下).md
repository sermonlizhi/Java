在[《依赖注入(上)》](https://blog.csdn.net/sermonlizhi/article/details/120748505)介绍了，会根据注入点来进行注入，而属性注入点和方法注入点的具体实现略有不同，但它们的核心方法都是通过ConfigurableListableBeanFactory的resolveDependency()类获取属性值，下面将先简单介绍一下属性注入点和方法注入点的逻辑，然后重点介绍resolveDependency()是如何工作

## 一、属性注入点

#### 1.1 调用注入方法

属性注入点调用的是AutowiredFieldElement.inject()，源码如下：

对于原型Bean，第一次创建的时候，也找注入点，然后进行注入，此时cached为false，注入完了之后cached为true，第二次创建的时候，先找注入点（此时会拿到缓存好的注入点），也就是AutowiredFieldElement对象，此时cache为true，也就进到此处了，注入点内并没有缓存被注入的具体Bean对象，而是beanName，这样就能保证注入到不同的原型Bean对象

```java
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {

   Field field = (Field) this.member;
   Object value;
   if (this.cached) {
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
   if (value != null) {
      ReflectionUtils.makeAccessible(field);
      field.set(bean, value);
   }
}
```

#### 1.2 获取属性注入值

如果缓存中没有，则直接调用resolveFieldValue()方法来获取注入值，该方法会将属性封装成一个依赖描述器对象，并且获取当前工厂的类型转换器，类型转换器在处理@Value注解时有很大用处，然后调用resolveDependency()方法来获取属性值，并把获取到属性值进行缓存

```java
private Object resolveFieldValue(Field field, Object bean, @Nullable String beanName) {

    DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
    desc.setContainingClass(bean.getClass());

    Set<String> autowiredBeanNames = new LinkedHashSet<>(1);
    TypeConverter typeConverter = beanFactory.getTypeConverter();
    Object value;
    value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
    synchronized (this) {
        if (!this.cached) {
            Object cachedFieldValue = null;
            ……
            // 缓存属性值
            this.cachedFieldValue = cachedFieldValue;
            this.cached = true;
        }
    }
    return value;
}
```

## 二、方法注入点

#### 2.1 调用注入方法

属性注入点调用的是AutowiredMethodElement.inject()，源码如下：

如果pvs中已经有当前注入点的值了，则跳过注入，然后同样的也是先判断缓存是否有注入值，如果有则从缓存取，如果没有则调用resolveMethodArguments()获取方法注入的参数值列表

```java
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable { 
   if (checkPropertySkipping(pvs)) {
      return;
   }
   Method method = (Method) this.member;
   Object[] arguments;
   if (this.cached) {
      arguments = resolveCachedArguments(beanName);
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

#### 2.2 获取方法参数的注入值

获取注入点方法的参数数量，生成一个依赖描述器的数组，存放所有参数的依赖描述，然后根据方法及参数下标索引，将参数生成一个MethodParameter对象，通过该对象生成参数的依赖描述，然后同样调用resolveDependency()来获取参数的注入值，然后将注入值存放在参数值列表中

获取方法参数的注入值放入数组之后，如果方法中的每个参数都是需要通过注入来进行赋值的话，则会将将对每个参数生成ShortcutDependencyDescriptor依赖描述的对象，存放在缓存里面

```java
private Object[] resolveMethodArguments(Method method, Object bean, @Nullable String beanName) {
    int argumentCount = method.getParameterCount();
    Object[] arguments = new Object[argumentCount];
    DependencyDescriptor[] descriptors = new DependencyDescriptor[argumentCount];
    Set<String> autowiredBeans = new LinkedHashSet<>(argumentCount);
    Assert.state(beanFactory != null, "No BeanFactory available");
    TypeConverter typeConverter = beanFactory.getTypeConverter();

    // 遍历每个方法参数，找到匹配的bean对象
    for (int i = 0; i < arguments.length; i++) {
        MethodParameter methodParam = new MethodParameter(method, i);

        DependencyDescriptor currDesc = new DependencyDescriptor(methodParam, this.required);
        currDesc.setContainingClass(bean.getClass());
        descriptors[i] = currDesc;
        Object arg = beanFactory.resolveDependency(currDesc, beanName, autowiredBeans, typeConverter);
        if (arg == null && !this.required) {
            arguments = null;
            break;
        }
        arguments[i] = arg;
    }
    
    synchronized (this) {
        if (!this.cached) {
            if (arguments != null) {
                DependencyDescriptor[] cachedMethodArguments = Arrays.copyOf(descriptors, arguments.length);
                registerDependentBeans(beanName, autowiredBeans);
                if (autowiredBeans.size() == argumentCount) {
                    // 缓存参数的依赖描述
                    ……
                }
                this.cachedMethodArguments = cachedMethodArguments;
            }
            this.cached = true;
        }
    }
    return arguments;
}
```

## 三、获取属性值

#### 3.1 resolveDependency()介绍

在上面的属性注入点和方法注入点进行注入时，最核心的逻辑就是如何获取依赖值，都是通过调用ConfigurableListableBeanFactory的resolveDependency()来实现，下面将介绍该方法的具体实现

resolveDependency()的源码如下，如果属性类型是Optional、ObjectFactory、ObjectProvider或javaxInjectProviderClass，那么会去调用指定的方法来获取依赖对象，但对于大多数情况，都会走最后面else里面的逻辑

```java
public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
      @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {
   // 用来获取方法入参名字的
   descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());

   // 所需要的类型是Optional
   if (Optional.class == descriptor.getDependencyType()) {
      return createOptionalDependency(descriptor, requestingBeanName);
   }
   // 所需要的的类型是ObjectFactory，或ObjectProvider
   else if (ObjectFactory.class == descriptor.getDependencyType() ||
         ObjectProvider.class == descriptor.getDependencyType()) {
      return new DependencyObjectProvider(descriptor, requestingBeanName);
   }
   else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
      return new Jsr330Factory().createDependencyProvider(descriptor, requestingBeanName);
   }
   else {
      // 在属性或set方法上使用了@Lazy注解，那么则构造一个代理对象并返回，真正使用该代理对象时才进行类型筛选Bean
      Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
            descriptor, requestingBeanName);

      if (result == null) {
         // descriptor表示某个属性或某个set方法
         // requestingBeanName表示正在进行依赖注入的Bean
         result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
      }
      return result;
   }
}
```

在resolveDependency()的第一行，会去初始化获取方法参数名的方式，getParameterNameDiscoverer()，该方法会返回默认的获取方法参数名的方式，在JDK1.8以下的版本中，没有办法通过反射来获取方法的参数名，Spring提供了通过本地变量表的方式(自行解析字节码文件)来获取方法的参数名，而JDK1.8中虽然提供了获取方法参数名的API，但返回的方法名并不是我们定义的那个参数名，需要在编译时指定”-parameters“参数才能拿到想要的参数名，而StandardReflectionParameterNameDiscoverer()的作用等同于”-parameters“

```java
public DefaultParameterNameDiscoverer() {
   // TODO Remove this conditional inclusion when upgrading to Kotlin 1.5, see https://youtrack.jetbrains.com/issue/KT-44594
   if (KotlinDetector.isKotlinReflectPresent() && !NativeDetector.inNativeImage()) {
      addDiscoverer(new KotlinReflectionParameterNameDiscoverer());
   }
   // 依次调用Discoverer来获取某个方法的参数名，反射（1.8）和本地变量表
   addDiscoverer(new StandardReflectionParameterNameDiscoverer());
   addDiscoverer(new LocalVariableTableParameterNameDiscoverer());
}
```

如果在属性或方法上使用了@Lazy注解指定懒加载，那么将会生成生成代理对象作为依赖值，当调用该对象方法的时候，会根据代理对象找到真正的Bean对象，然后调用Bean对象的方法getLazyResolutionProxyIfNecessary()会去调用ContextAnnotationAutowireCandidateResolver.buildLazyResolutionProxy()

```java
Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
    descriptor, requestingBeanName);

if (result == null) {
    // descriptor表示某个属性或某个set方法
    // requestingBeanName表示正在进行依赖注入的Bean,解析@Lazy注解生成Bean时也会调用该方法
    result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
}
return result;
```

@Lazy注解标识的属性获取注入值的逻辑如下，与没有@Lazy注解一样，都会去调用DefaultListableBeanFactory.doResolveDependency()来生成一个原始的Bean实例，然后根据生成的Bean实例生成一个代理Bean，最后返回的也是代理Bean

```java
protected Object buildLazyResolutionProxy(final DependencyDescriptor descriptor, final @Nullable String beanName) {
	……
    final DefaultListableBeanFactory dlbf = (DefaultListableBeanFactory) beanFactory;
    TargetSource ts = new TargetSource() {
        ……
        @Override
        public Object getTarget() {
            Set<String> autowiredBeanNames = (beanName != null ? new LinkedHashSet<>(1) : null);
            Object target = dlbf.doResolveDependency(descriptor, beanName, autowiredBeanNames, null);
            
            if (autowiredBeanNames != null) {
                for (String autowiredBeanName : autowiredBeanNames) {
                    if (dlbf.containsBean(autowiredBeanName)) {
                        dlbf.registerDependentBean(autowiredBeanName, beanName);
                    }
                }
            }
            return target;
        }
        ……
    };

    ProxyFactory pf = new ProxyFactory();
    pf.setTargetSource(ts);
    Class<?> dependencyType = descriptor.getDependencyType();
    if (dependencyType.isInterface()) {
        pf.addInterface(dependencyType);
    }
    return pf.getProxy(dlbf.getBeanClassLoader());
}
```

#### 3.2 详解doResolveDependency()

前面resolveDependency()方法中，根据属性类型以及是否有@Lazy注解，提供了不同获取Bean实例的方法，但是，不论是哪种获取Bean的方法，其内部的核心实现都是通过调用DefaultListableBeanFactory.doResolveDependency()来完成的，下面将重点介绍该方法的源码实现

首先判断当前需要注入的依赖，已经在别处做过依赖注入了，则无需再生成依赖值，可以直接从缓存中取

比如UserService实例中依赖了User实例，而OrderService实例中也依赖User实例，但OrderService实例先创建，在OrderService实例的创建过程中已经做过了User的依赖注入，那么当UserService注入User的时候，就无需再匹配各种条件去找了，直接从缓存中取

```java
// 如果当前descriptor之前做过依赖注入了，则可以直接取shortcut了，相当于缓存
Object shortcut = descriptor.resolveShortcut(this);
if (shortcut != null) {
    return shortcut;
}
```

##### 3.2.1 处理@Value

@Value可以用于成员变量、方法以及方法参数上，对其进行依赖注入时，首先去获取value注解指定的值，@Value所指定值，可以是一个普通的字符串，也可以是一个占位符，或者是一个Spring表达式，所以当获取到value指定的值后，需要对其进行各种处理

```java
Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
if (value != null) {
    if (value instanceof String) {
        // 占位符填充(${})
        String strVal = resolveEmbeddedValue((String) value);
        BeanDefinition bd = (beanName != null && containsBean(beanName) ?
                             getMergedBeanDefinition(beanName) : null);
        // 解析Spring表达式(#{})
        value = evaluateBeanDefinitionString(strVal, bd);
    }
    // 将value转化为descriptor所对应的类型
    TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
    try {
        return converter.convertIfNecessary(value, type, descriptor.getTypeDescriptor());
    }
    catch (UnsupportedOperationException ex) {
        // A custom TypeConverter which does not support TypeDescriptor resolution...
        return (descriptor.getField() != null ?
                converter.convertIfNecessary(value, type, descriptor.getField()) :
                converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
    }
}
```

- 获取@Value所指定的值

  调用QualifierAnnotationAutowireCandidateResolver.getSuggestedValue()来解析@Value，由于@Value可以用于变量(成员变量和方法参数)和方法上，首先去查看变量上的@Value值，descriptor.getAnnotations()首先去判断是不是成员变量，如果不是则取方法参数的注解

  如果变量都没有value注解，再去找方法上的value注解值

  ```java
  public Object getSuggestedValue(DependencyDescriptor descriptor) {
     Object value = findValue(descriptor.getAnnotations());
     if (value == null) {
        MethodParameter methodParam = descriptor.getMethodParameter();
        if (methodParam != null) {
           value = findValue(methodParam.getMethodAnnotations());
        }
     }
     return value;
  }
  ```

- @Value所指定的值解析

  @Value的值可能是占位符或Spring表达式，需要将其解析的到具体的值，首先解析占位符，resolveStringValue()是一个函数式接口的方法，循环遍历所有的占位符解析器，调用resolveStringValue()

  ```java
  public String resolveEmbeddedValue(@Nullable String value) {
     if (value == null) {
        return null;
     }
     String result = value;
     for (StringValueResolver resolver : this.embeddedValueResolvers) {
        result = resolver.resolveStringValue(result);
        if (result == null) {
           return null;
        }
     }
     return result;
  }
  ```

  在BeanFactory初始化的过程中，有如下一段代码，主要就是从配置的系统变量获取对应的值，Spring的Environment包含了计算机所有的系统环境变量、运行时指定的变量以及Spring的配置文件中的配置信息，解析占位符时会把"${}"里面的字符串当作Key来查找值，如果找不到，就会把整个@Value所指定的值当作它的值

  ```java
  // 设置默认的占位符解析器  ${xxx}  ---key
  if (!beanFactory.hasEmbeddedValueResolver()) {
     beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
  }
  ```

  然后进行Spring表达式的解析，Spring表达式支持两个写法，一种是直接@Value指定"#{}"格式，里面的字符串就是Spring表达式；另一种写法是可以将通过占位符来配置Spring表达式，"${}"里面字符串是表达式的Key，对应的配置的值就是Spring表达式

- 类型转换

  解析获取到@Value的真实值后，如果真实值的类型，与依赖值的类型不匹配，则需要进行类型转换，首先需要找到能对当前类型和目标类型做转换的转换器，如果找不到则会抛异常，如果找到了则进行转换，然后将转换后的值作为依赖注入的依赖值

  如下代码所示，使用@Value注解来注入User对象，因为Spring没有提供将String类型转换为User类型的转换器，所以直接运行以下代码将会报错，我们可以通过自定义类型转换器，来实现属性注入

  ```java
  @Component
  public class UserService {
  
     @Value("lizhi")
     User user;
  
     public void test() {
        System.out.println(user);
     }
  }
  ```

  自定义类型转换器，需要实现Converter接口，source就是@Value中的指定的值

  ```java
  public class CustomizedTypeConvert implements Converter {
     @Override
     public Object convert(Object source) {
        User user = new User(source.toString());
        return user;
     }
  }
  ```

##### 3.2.2 注入多个同类实例

如果没有@Value注解，则会注入Spring中的实例

Spring不仅支持注入一个实例，还支持注入多个同类实例，比如：

```java
@Autowired
Map<String,User> userMap;

@Autowired
List<User> userList;
```

如果依赖所对应的类型是数组、Map这些，就将依赖对应的类型所匹配的所有bean方法，不用进一步做筛选了

```java
Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
if (multipleBeans != null) {
   return multipleBeans;
}
```

resolveMultipleBeans()方法中，Spring5.3.10版本支持Stream、数组、集合和Map四种类型，而对于Map类型，它的Key必须是String类型，Value类型不能是null，否则将直接返回Null，不论是哪一种实现，都会去调用findAutowireCandidates()来获取匹配Bean实例

##### 3.2.3 匹配满足条件的Bean实例或beanClass

找到所有Bean，key是beanName, value有可能是bean对象，有可能是beanClass，因为有些Bean是懒加载的，此时Spring容器中还没有该Bean的实例，对于懒加载的Bean，我们只能获取到beanClass，然后在进行对其进行实例化

```java
Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
if (matchingBeans.isEmpty()) {
   // required为true，抛异常
   if (isRequired(descriptor)) {
      raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
   }
   return null;
}
```

findAutowireCandidates()就是去匹配所有满足条件的Bean，具体实现如下：

- 查找BeanName

  从BeanFactory中找出和requiredType所匹配的beanName，仅仅是beanName，这些bean不一定经过了实例化，只有到最终确定某个Bean了，如果这个Bean还没有实例化才会真正进行实例化

  ```java
  String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
        this, requiredType, true, descriptor.isEager());
  Map<String, Object> result = CollectionUtils.newLinkedHashMap(candidateNames.length);
  ```

- 从缓存中匹配Bean

  根据类型从resolvableDependencies中匹配Bean，resolvableDependencies中存放的是类型：Bean对象，比如BeanFactory.class:BeanFactory对象，在Spring启动时设置

  匹配到相同的类型之后，取出Bean实例，然后判断是不是requiredType的实例，如果是则加入到结果集中

  ```java
  for (Map.Entry<Class<?>, Object> classObjectEntry : this.resolvableDependencies.entrySet()) {
     Class<?> autowiringType = classObjectEntry.getKey();
     if (autowiringType.isAssignableFrom(requiredType)) {
        Object autowiringValue = classObjectEntry.getValue();
        autowiringValue = AutowireUtils.resolveAutowiringValue(autowiringValue, requiredType);
  
        if (requiredType.isInstance(autowiringValue)) {
           result.put(ObjectUtils.identityToString(autowiringValue), autowiringValue);
           break;
        }
     }
  }
  ```

- 判断能不能进行注入

  首先判断需要注入的实例是不是自己(Spring支持注入自己)，然后判断当前依赖的beanName是否可以用来进行自动注入，如果当前beanName既不是自己也可以进行依赖注入，则

  ```java
  for (String candidate : candidateNames) {
     // 如果不是自己，则判断该candidate到底能不能用来进行自动注入
     if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, descriptor)) {
        addCandidateEntry(result, candidate, descriptor, requiredType);
     }
  }
  ```

  判断是不是自己，就根据beanName来判断，先判断普通的beanName，再判断factoryBeanName，Spring注入的时候，先找不是自己的Bean实例，如果找不到，再把自己注入进来

  ```java
  private boolean isSelfReference(@Nullable String beanName, @Nullable String candidateName) {
     return (beanName != null && candidateName != null &&
           (beanName.equals(candidateName) || (containsBeanDefinition(candidateName) &&
                 beanName.equals(getMergedLocalBeanDefinition(candidateName).getFactoryBeanName()))));
  }
  ```

  判断当前beanName能都进行注入也很简单，根据BeanDefinition判断beanName对应的Bean可不可以用来进行依赖注入

  ```java
  protected boolean isAutowireCandidate(
      String beanName, DependencyDescriptor descriptor, AutowireCandidateResolver resolver)
      throws NoSuchBeanDefinitionException {
  
      // 根据BeanDefinition判断beanName对应的Bean可不可以用来进行依赖注入
      String bdName = BeanFactoryUtils.transformedBeanName(beanName);
      if (containsBeanDefinition(bdName)) {
          return isAutowireCandidate(beanName, getMergedLocalBeanDefinition(bdName), descriptor, resolver);
      }
      else if (containsSingleton(beanName)) {
          return isAutowireCandidate(beanName, new RootBeanDefinition(getType(beanName)), descriptor, resolver);
      }
      BeanFactory parent = getParentBeanFactory();
      //递归调用
      ……
  }
  ```

  上述条件都满足之后，获取注入值，加入的结果集中，如果注入的是数组、集合和Map三种类型，直接调用getBean去创建，否则从单例池中去取，如果没有，将beanClass存一下，后面再实例化

  ```java
  private void addCandidateEntry(Map<String, Object> candidates, String candidateName,
        DependencyDescriptor descriptor, Class<?> requiredType) {
  
     if (descriptor instanceof MultiElementDescriptor) {
        Object beanInstance = descriptor.resolveCandidate(candidateName, requiredType, this);
        if (!(beanInstance instanceof NullBean)) {
           candidates.put(candidateName, beanInstance);
        }
     }
     else if (containsSingleton(candidateName) || (descriptor instanceof StreamDependencyDescriptor &&
           ((StreamDependencyDescriptor) descriptor).isOrdered())) {
        // 如果在单例池中存在，则直接放入bean对象
        Object beanInstance = descriptor.resolveCandidate(candidateName, requiredType, this);
        candidates.put(candidateName, (beanInstance instanceof NullBean ? null : beanInstance));
     }
     else {
        // 将匹配的beanName，以及beanClass存入
        candidates.put(candidateName, getType(candidateName));
     }
  }
  ```

- 注入自己

  前面进行诸如判断的时候，优先其他的Bean实例，如果没有满足条件的Bean，判断是不是要注入自己，如果要注入自己，则保存自己的beanClass

  ```java
  // 为空要么是真的没有匹配的，要么是匹配的自己
  if (result.isEmpty()) {
     // 需要匹配的类型是不是Map、数组之类的
     boolean multiple = indicatesMultipleBeans(requiredType);
     // 匹配的是自己，被自己添加到result中
     if (result.isEmpty() && !multiple) {
        // Consider self references as a final pass...
        // but in the case of a dependency collection, not the very same bean itself.
        for (String candidate : candidateNames) {
           if (isSelfReference(beanName, candidate) &&
                 (!(descriptor instanceof MultiElementDescriptor) || !beanName.equals(candidate)) &&
                 isAutowireCandidate(candidate, fallbackDescriptor)) {
              addCandidateEntry(result, candidate, descriptor, requiredType);
           }
        }
     }
  }
  ```

如果没有找到匹配的Bean实例和beanClass，判断注解上的required值是否为true，如果为true，表示必须要进行注入，则将抛出异常

##### 3.2.3  筛选出最终的Bean实例或beanClass

如果根据类型匹配到了多个Bean实例或beanClass，进一步筛选出某一个, @Primary-->优先级最高--->name，如果得到的是一个beanClass，则进行getBean()进行实例化

```java
if (matchingBeans.size() > 1) {
    // 根据类型找到了多个Bean，进一步筛选出某一个, @Primary-->优先级最高--->name
    autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
    instanceCandidate = matchingBeans.get(autowiredBeanName);
}

// 有可能筛选出来的是某个bean的类型，此处就进行实例化，调用getBean
if (instanceCandidate instanceof Class) {
    instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
}
```

## 四、核心流程图

#### 4.1 根据Type获取BeanName流程图

<img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/Spring%E6%BA%90%E7%A0%81.jpg" alt="Spring源码" style="zoom:50%;" />

#### 4.2 判断Bean是否可以用来依赖注入流程图

<img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5-%E5%88%A4%E6%96%ADBean%E6%98%AF%E5%90%A6%E5%8F%AF%E4%BB%A5%E6%B3%A8%E5%85%A5.jpg" style="zoom:50%;" />
