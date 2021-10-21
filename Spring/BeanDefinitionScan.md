##### Spring容器在创建Bean之前，需要扫描指定包下的文件，然后生成BeanDifinition，下面将介绍Spring是如何进行扫描，然后再生成BeanDifinition

#### 1、scan方法的入参是字符串数组,可以同时指定多个包进行扫描,调用doScan方法来进行扫描

```java
public class ClassPathBeanDefinitionScanner{
	public int scan(String... basePackages) {
		int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

		doScan(basePackages);

		// Register annotation config processors, if necessary.
		if (this.includeAnnotationConfig) {
			AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
		}

		return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
	}
}
```



#### 2、调用findCandidateComponents方法，生成BeanDefinition

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
		for (String basePackage : basePackages) {

			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
            ……
        }
}
```

findCandidateComponents方法中，Spring支持两种获取类名的方式：

```java
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
    if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
        // 指定的类
        return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
    }
    else {
        // 扫描指定包路径下的所有类
        return scanCandidateComponents(basePackage);
    }
}
```

- 在resources/META-INF/spring.components文件中指定

  ```java
  // 类名=注解名
  com.lizhi.service.UserService=org.springframework.stereotype.Component
  ```

- 扫描指定包下面的所有类

`上面两种方式只是获取类名的方式不同,根据类生成BeanDefinition的逻辑是完全相同的`



#### 3、获取指定路径下所有的.class文件

```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
    Set<BeanDefinition> candidates = new LinkedHashSet<>();
    try {
        // 获取basePackage下所有的文件资源
        String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
            resolveBasePackage(basePackage) + '/' + this.resourcePattern;
        Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
        for (Resource resource : resources) {
				if (resource.isReadable()) {
					try {
                        // 元数据读取器
						MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
						// 下面根据类的元数据判断是否可以生成BeanDefinition
                        ……
                    }
                }
        }
        ……
    }
}                        
```

Spring提供的元数据读取器可以获取类的元数据，比如类名、类中的方法、类上的注解，元数据读取器的默认实现为**SimpleMetadataReader**

`Spring中元数据读取器解析类时使用的是ASM技术,并不需要将class文件都加载进内存进行解析;JVM规范对class文件的加载是按需加载,不能违背该规范`



#### 4、ComponentScan注解可以指定排除过滤器和包含过滤器，根据注册的过滤器判断是否需要生成BeanDefinition

```java
// excludeFilters、includeFilters判断
if (isCandidateComponent(metadataReader)) {
    …………
}
```

首先判断当前类是否被排除，如果与某个排除过滤器匹配则不能生成BeanDefinition，然后再判断是否能匹配到包含过滤器

```java
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
    for (TypeFilter tf : this.excludeFilters) {
        if (tf.match(metadataReader, getMetadataReaderFactory())) {
            return false;
        }
    }

    // 符合includeFilters的会进行条件匹配，也就是先看有没有@Component，再看是否符合@Conditional
    for (TypeFilter tf : this.includeFilters) {
        if (tf.match(metadataReader, getMetadataReaderFactory())) {
            return isConditionMatch(metadataReader);
        }
    }
    return false;
}
```

`以AnnotationConfigApplicationContext为例,Spring容器在启动的时候,会创建BeanDefinitionScanner扫描器,在创建Scnner的时候会注册一个Component的默认包含过滤器`

```java
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean 			        useDefaultFilters, Environment environment, @Nullable ResourceLoader                      resourceLoader) {
    // 注册默认包含过滤器
    if (useDefaultFilters) {
        registerDefaultFilters();
    }
}
```

```java
protected void registerDefaultFilters() {

    // 注册@Component对应的AnnotationTypeFilter
    this.includeFilters.add(new AnnotationTypeFilter(Component.class));
}
```



#### 5、条件匹配

如果一个类有对应的包含过滤器它可能还不能生成BeanDefinition，还需要判断条件注解是否满足

```java
for (TypeFilter tf : this.includeFilters) {
    if (tf.match(metadataReader, getMetadataReaderFactory())) {
        return isConditionMatch(metadataReader);
    }
}
```

条件鉴定器根据类上的注解来进行匹配

```java
private boolean isConditionMatch(MetadataReader metadataReader) {
    if (this.conditionEvaluator == null) {
        this.conditionEvaluator =
            new ConditionEvaluator(getRegistry(), this.environment, this.resourcePatternResolver);
    }
    // 参数为该类的所有注解
    return !this.conditionEvaluator.shouldSkip(metadataReader.getAnnotationMetadata());
}
```

如果没有Conditional注解返回false，isConditionMatch将返回true，如果有Conditional注解则判断是否满足条件

```java
public boolean shouldSkip(@Nullable AnnotatedTypeMetadata metadata, @Nullable ConfigurationPhase phase) {
    if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
        return false;
    }
    …………
}
```



#### 6、生成BeanDefinition

经过了过滤器和条件鉴定器之后，生成BeanDefinition，这个时候BeanDefinition的beanClass属性是beanClassName，而不是Class对象，因为此时只是扫描，还没进行加载

还需要判断这个BeanDefinition是否可以实例化成一个对象

```java
if (isCandidateComponent(metadataReader)) {
    ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
    sbd.setSource(resource);

    if (isCandidateComponent(sbd)) {
        candidates.add(sbd);
    }
}
```

下面代码的isIndependent()判断当前类是否是独立的，对于一个普通的内部类来说，编译后它也是一个独立的class文件，但它的实例化依赖顶级类，而顶级类和静态内部类可以直接实例化

isConcrete()判断该类是否是接口或抽象类，接口和抽象类编译后也是一个class文件，但它们都不能进行实例化

如果该类是一个抽象类，但是有Lookup注解定义的方法，那么也可以进行实例化

```java
protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
    AnnotationMetadata metadata = beanDefinition.getMetadata();
    return (metadata.isIndependent() && (metadata.isConcrete() ||
                                         (metadata.isAbstract() && metadata.hasAnnotatedMethods(Lookup.class.getName()))));
}
```

**Lookup注解的用途：**

```java
@Component
@Scope("prototype")
public class User {

}

@Component
public class UserService {

	@Autowired
	User user;

	public void test() {
		System.out.println(user);
	}

}
```

上面的代码，虽然User是一个原型Bean，但是每次调用UserService的test()，打印的都是同一个User对象，因为UserService是单例的，但经过下面Lookup注解，就可以得到不同的User对象

```java
@Component
public class UserService {

	@Autowired  //有没有该注解都无所谓
	User user;

	public void test() {
		System.out.println(createUser());
	}

	@Lookup("user")
	public User createUser(){
		return null;
	}
}
```



#### 7、填充扫描生成BeanDefinition的各个属性

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    //扫描生成BeanDefinition
    Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
    for (BeanDefinition candidate : candidates) {
        //填充Scope属性值,如果注解Scope有值,用该值填充
        ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
        candidate.setScope(scopeMetadata.getScopeName());
		
        //生成beanName
        String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);

        //设置默认值以及是否可以作为依赖注入
        if (candidate instanceof AbstractBeanDefinition) {
            postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
        }
        if (candidate instanceof AnnotatedBeanDefinition) {
            // 解析@Lazy、@Primary、@DependsOn、@Role、@Description
            AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
        }

        // 检查Spring容器中是否已经存在该beanName
        if (checkCandidate(beanName, candidate)) {
            BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
            definitionHolder =
                AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
            beanDefinitions.add(definitionHolder);

            // 注册
            registerBeanDefinition(definitionHolder, this.registry);
        }
    }
    return beanDefinitions;
}
```

- ##### 获取beanName

  调用AnnotationBeanNameGenerator的generateBeanName()方法来生成beanName，如果是扫描生成的BeanDefinition，则判断注解有没有指定beanName

  ```java
  public String generateBeanName(BeanDefinition definition, BeanDefinitionRegistry registry) {
      if (definition instanceof AnnotatedBeanDefinition) {
          // 获取注解所指定的beanName
          String beanName = determineBeanNameFromAnnotation((AnnotatedBeanDefinition) definition);
          if (StringUtils.hasText(beanName)) {
              // Explicit bean name found.
              return beanName;
          }
      }
      // Fallback: generate a unique default bean name.
      return buildDefaultBeanName(definition, registry);
  }
  ```

  如果注解没有指定，则调用buildDefaultBeanName()生成默认的beanName

  ```java
  protected String buildDefaultBeanName(BeanDefinition definition) {
      //扫描的时候如果没有指定className,将会抛出异常
      String beanClassName = definition.getBeanClassName();
      Assert.state(beanClassName != null, "No bean class name set");
      String shortClassName = ClassUtils.getShortName(beanClassName);
      return Introspector.decapitalize(shortClassName);
  }
  ```

  如果类名的前两个字符都是大写，则直接将类名作为beanName，否则将类名首字符变成小写后将作为beanName

  ```java
  public static String decapitalize(String name) {
      if (name == null || name.length() == 0) {
          return name;
      }
      if (name.length() > 1 && Character.isUpperCase(name.charAt(1)) &&
          Character.isUpperCase(name.charAt(0))){
          return name;
      }
      char chars[] = name.toCharArray();
      chars[0] = Character.toLowerCase(chars[0]);
      return new String(chars);
  }
  ```

- 设置默认值

  ```java
  if (candidate instanceof AbstractBeanDefinition) {
      postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
  }
  ```

  ```java
  protected void postProcessBeanDefinition(AbstractBeanDefinition beanDefinition, String beanName) {
      // 设置BeanDefinition的默认值
      beanDefinition.applyDefaults(this.beanDefinitionDefaults);
  
      // AutowireCandidate表示某个Bean能否被用来做依赖注入
      if (this.autowireCandidatePatterns != null) {
          beanDefinition.setAutowireCandidate(PatternMatchUtils.simpleMatch(this.autowireCandidatePatterns, beanName));
      }
  }
  ```

  设置默认值

  ```java
  public void applyDefaults(BeanDefinitionDefaults defaults) {
      Boolean lazyInit = defaults.getLazyInit();
      if (lazyInit != null) {
          setLazyInit(lazyInit);
      }
      setAutowireMode(defaults.getAutowireMode());
      setDependencyCheck(defaults.getDependencyCheck());
      setInitMethodName(defaults.getInitMethodName());
      setEnforceInitMethod(false);
      setDestroyMethodName(defaults.getDestroyMethodName());
      setEnforceDestroyMethod(false);
  }
  ```

- 检查Spring容器是否已经有该beanName

  ```java
  if (checkCandidate(beanName, candidate)) {
      BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
      definitionHolder =
          AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
      beanDefinitions.add(definitionHolder);
  
      // 注册 --放入到registry的BeanfinitionMap中
      registerBeanDefinition(definitionHolder, this.registry);
  }
  ```

  如果BeanfinitionMap中不存在当前beanName，则返回True，生成一个新的Beanfinition注册到BeanfinitionMap

  如果存在beanName，则从BeanfinitionMap中根据beanName取出Beanfinition，与当前的Beanfinition比较，判断是否相等，如果相等说明同一个类被扫描了两次，无需再向BeanfinitionMap中添加，如果不相等，则直接抛出异常

  ```java
  protected boolean checkCandidate(String beanName, BeanDefinition beanDefinition) throws IllegalStateException {
      if (!this.registry.containsBeanDefinition(beanName)) {
          return true;
      }
      BeanDefinition existingDef = this.registry.getBeanDefinition(beanName);
      BeanDefinition originatingDef = existingDef.getOriginatingBeanDefinition();
      if (originatingDef != null) {
          existingDef = originatingDef;
      }
      // 是否兼容，如果兼容返回false表示不会重新注册到Spring容器中，如果不冲突则会抛异常。
      if (isCompatible(beanDefinition, existingDef)) {
          return false;
      }
      throw new ConflictingBeanDefinitionException("Annotation-specified bean name '" + beanName +
                                                   "' for bean class [" + beanDefinition.getBeanClassName() + "] conflicts with existing, " +
                                                   "non-compatible bean definition of same name and class [" + existingDef.getBeanClassName() + "]");
  }
  ```



#### 8、spring.components

前面提到过调用findCandidateComponents()生成BeanDefinition的时候，Spring提供了两种方式，前面介绍的方式是扫描整个包路径下的文件，Spring还提供了配置文件的方式，直接在META-INF/spring.components中指定哪些类需要生成Bean

```java
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
	//根据spring.components生成BeanDefinition
    if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
        return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
    }
    else {
        return scanCandidateComponents(basePackage);
    }
}
```

spring.components文件配置：

```java
// 类名=注解名 结构
com.lizhi.service.UserService=org.springframework.stereotype.Component
```



在Spring容器启动创建扫描器的时候，ClassPathBeanDefinitionScanner类的ClassPathBeanDefinitionScanner()就会加载spring.components文件信息

```java
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
                                      Environment environment, @Nullable ResourceLoader resourceLoader) {

    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    this.registry = registry;

    if (useDefaultFilters) {
        registerDefaultFilters();
    }
    setEnvironment(environment);
    //加载资源
    setResourceLoader(resourceLoader);
}
```

```java
public void setResourceLoader(@Nullable ResourceLoader resourceLoader) {
    this.resourcePatternResolver = ResourcePatternUtils.getResourcePatternResolver(resourceLoader);
    this.metadataReaderFactory = new CachingMetadataReaderFactory(resourceLoader);
    //加载spring.components文件信息
    this.componentsIndex = CandidateComponentsIndexLoader.loadIndex(this.resourcePatternResolver.getClassLoader());
}
```

`扫描spring.components生成的原始数据存放在CandidateComponentsIndex.index中，index是Spring实现的一个MultiValueMap,MultiValueMap的key,可以对应多个value值.index的key为注解名称,value为类名,也就是在spring.components文件中,相同注解的类名存放在一个list里面`

- addCandidateComponentsFromIndex()实现

  extractStereotype()方法获取包含过滤器指定的注解名称，可以是Comopnent注解，也可以是其他注解

  ```java
  private Set<BeanDefinition> addCandidateComponentsFromIndex(CandidateComponentsIndex index, String basePackage) {
      Set<BeanDefinition> candidates = new LinkedHashSet<>();
      try {
          //类名
          Set<String> types = new HashSet<>();
          for (TypeFilter filter : this.includeFilters) {
              // Component注解
              String stereotype = extractStereotype(filter);
              if (stereotype == null) {
                  throw new IllegalArgumentException("Failed to extract stereotype from " + filter);
              }
              types.addAll(index.getCandidateTypes(basePackage, stereotype));
          }
          //后面的代码同全包扫描一样，遍历类名，生成满足条件的BeanDefinition
          ……
  }
  ```

  根据注解的名称从CandidateComponentsIndex中取出与当前包名匹配的类名，放入到types中，types中存放的是类的完整名称(包含路径)

  ```java
  // stereotype=注解名
  public Set<String> getCandidateTypes(String basePackage, String stereotype) {
      //根据注解取出所有定义该注解的实体(包含包名和类名)
      List<Entry> candidates = this.index.get(stereotype);
      if (candidates != null) {
          //匹配包名,返回类名的集合
          return candidates.parallelStream()
              .filter(t -> t.match(basePackage))
              .map(t -> t.type)
              .collect(Collectors.toSet());
      }
      return Collections.emptySet();
  }
  
  private static class Entry {
  
      private final String type;
  
      private final String packageName;
      ……
  }
  ```

  



