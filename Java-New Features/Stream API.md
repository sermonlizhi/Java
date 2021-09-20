## 一、Stream简介

#### 1.1 Stream API说明

- Stream API(java.util.stream)把真正的函数式编程风格引入到Java中，提供了简单高效的操作
- Stream 是Java8中处理集合的关键抽象概念，它可以指定对集合进行操作，可以执行非常复杂的查找、过滤和映射数据等操作。使用Stream API对集合数据进行操作，就类似于使用SQL执行的数据库查询。也可以使用Stream API来并行执行操作

#### 1.2 什么是Stream

Stream是数据渠道，用于操作数据源(集合、数组等)所生成的元素序列。集合存储数据，Stream是对集合元素进行计算

- Stream本省并不存储元素
- Stream不会改变源对象，会返回一个持有结果的新Stream
- Stream是延迟执行的，只有等到需要结果的时候才执行

#### 1.3 Stream操作步骤

- 创建Stream

  一个数据源(如：集合、数组)获取一个流

- 中间操作

  一个中间操作链，可以进行多种操作，对数据源的数据进行处理

- 终止操作

  一旦执行终止操作，就执行中间操作链，并产生结果，之后，创建的流将不能再被使用，如果使用，需要重新创建一个新的流

<img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210920092018265.png" alt="image-20210920092018265" style="zoom:50%;" />

## 二、Stream使用

#### 2.1 创建执行流

- 集合创建执行流

  ```java
  List<User> userList = User.getUserList();
  //创建顺序流
  Stream<User> userStream = userList.stream();
  //创建并行流 中间操作可以并行执行
  Stream<User> userParallelStream = userList.parallelStream();
  ```

  `Collection接口中实现了默认的stream和parallelStream方法,所以Collection的实现类可以直接调用这两个方法`

- 通过数组创建执行流

  ```java
  int[] intArr = {1,2,3,4,5,6};
  IntStream intStream = Arrays.stream(intArr);
  
  int[] longArr = {1,2,3,4,5,6};
  IntStream longStream = Arrays.stream(longArr);
  
  double[] doubleArr = {1.1,2.2,3.3};
  DoubleStream stream = Arrays.stream(doubleArr);
  
  User[] userArr = {new User(1,"zhangsan",25),new User(2,"lisi",26)};
  Stream<User> userStream = Arrays.stream(userArr);
  ```

  `Arrays工具类中提供了静态的stream方法实现,其中基本类型int、long、double的数组返回的是对应与基本类型的执行流`

- 通过Stream的of方法创建

  ```java
  Stream<Integer> integerStream = Stream.of(1, 2, 3, 4, 5, 6);
  Stream<Boolean> booleanStream = Stream.of(true, false);
  Stream<Float> floatStream = Stream.of(1.1f, 2.2f, 3.3f);
  ```

  `通过of方法创建基本数据类型的执行流时,返回的基本类型对应的包装类型的执行流`

- 通过Stream的iterate()和generate()创建无限流

  ```java
  Stream.iterate(0,t -> t+2).limit(10).forEach(System.out::println);
  
  Stream.generate(Math::random).limit(10).forEach(System.out::println);
  ```

  `iterate是通过迭代的方式一直生成数据,其中第一个参数为起始值,后面的lambda表达式为具体的迭代逻辑,如果没有通过后面的limit方法限制,则会一直迭代下去`

  `generate是通过重复调用lambda表达式来生成数据,如果不加limit限制,也会一直调用里面的lambda表达式`

#### 2.2 Stream的中间操作

Stream的中间操作可以连接起来形成一个调用链(流水线)，只有流水线执行了终止操作，中间操作才会执行，否则中间操作不会做任何处理，而是在终止操作时一次性处理，这种方式称为“惰性求值”

- 筛选与切片

  ```java
  //过滤出年龄大于25的用户,filter的参数是Predicate函数接口的实现
  List<User> userList = User.getUserList();
  userList.stream().filter(user -> user.getAge() >25).forEach(System.out::println);
  ```

  ```java
  //限制集合元素的数量,只输出三个用户
  List<User> userList = User.getUserList();
  userList.stream().limit(3).forEach(System.out::println);
  ```

  ```java
  //跳过前面指令数量的集合元素,例子中为跳过前两个
  List<User> userList = User.getUserList();
  userList.stream().skip(2).forEach(System.out::println);
  ```

  ```java
  //distinct去重,通过流所生成元素的hashCode()和equals()去除重复元素
  List<User> userListDistinct = new ArrayList<User>(){
      {
          new User(1,"lizhi",23);
          new User(1,"lizhi",23);
          new User(1,"lizhi",23);
      }
  };
  userListDistinct.stream().distinct().forEach(System.out::println);
  ```

  `如果Stream流已经执行过了终止操作,再次调用该流的Stream API将会导致异常:java.lang.IllegalStateException: stream has already been operated upon or closed`

- 映射

  ```java
  //通过映射原有Stream的元素映射成新的元素构成的Stream
  List<String> list = Arrays.asList("aa","bb","cc");
  list.stream().map(String::toUpperCase).forEach(System.out::println);
  
  //将user实体集合映射成user的name集合
  List<User> userList = User.getUserList();
  Stream<String> nameStream = userList.stream().map(User::getName);
  nameStream.filter(name -> name.startsWith("li")).forEach(System.out::println);
  
  //flatmap接收一个函数作为参数,将流中的每个值都映射成另一个流,然后将所有流合并成一个流
  Stream<Character> characterStream = list.stream().flatMap(StreamOperateTest::stringFormat);
  characterStream.forEach(System.out::println);
  /** 原始写法 */
  Stream<Stream<Character>> streamStream = list.stream().map(StreamOperateTest::stringFormat);
  streamStream.forEach(stream -> stream.forEach(System.out::println));
  
  public static Stream<Character> stringFormat(String string){
      List<Character> list = new ArrayList<>(8);
      for (Character c: string.toCharArray()) {
          list.add(c);
      }
      return list.stream();
  }
  ```

  `映射将原有的Stream流通过映射关系转换成一个新的Stream流`

- 排序

  ```java
  //默认排序
  List<Integer> integerList = Arrays.asList(12, 95, 32, 2, 95, -2, 35);
  integerList.stream().sorted().forEach(System.out::println);
  
  //自定义排序
  List<User> userList = User.getUserList();
  userList.stream().sorted(Comparator.comparingInt(User::getAge)).forEach(System.out::println);
  ```

#### 2.3 终止操作

- 查找与匹配

  ```java
  //检查是否匹配所有元素
  List<User> userList = User.getUserList();
  boolean allMatch = userList.stream().allMatch(user -> user.getAge() > 24);
  
  //检查是否匹配任何一个元素
  boolean anyMatch = userList.stream().anyMatch(user -> user.getAge() > 25);
  
  //检查是否没有匹配的元素
  boolean noneMatch = userList.stream().noneMatch(user -> user.getAge() > 25);
  
  //查找第一个元素
  Optional<User> firstUser = userList.stream().findFirst();
  
  //查找任意一个元素
  Optional<User> anyUser = userList.stream().findAny();
  
  //统计流中元素个数
  long count = userList.stream().count();
  
  //统计流中最大值  如果比较的用户最大年龄有多个时,则任意返回一个
  Optional<User> maxUser = userList.stream().max(Comparator.comparing(User::getAge));
  ```

- 归约

  ```java
  //将流中元素反复结合起来,得到一个值
  //计算集合中元素的和  第一个参数为起始值,也参与运算
  List<Integer> integerList = Arrays.asList(5, 6, 8, 22, 72, 365, 85);
  Integer reduce = integerList.stream().reduce(0, Integer::sum);
  
  //不需要初始值反复运算,通过map将User流变成age流
  Optional<Integer> allAge = userList.stream().map(User::getAge).reduce(Integer::sum);
  ```

- 收集

  ```java
  //将操作后的数据收集到一个集合中
  List<User> userList = User.getUserList();
  
  List<User> collect = userList.stream().filter(user -> 
                                                user.getAge() > 25).collect(Collectors.toList());
  Set<User> userSet = userList.stream().filter(user -> 
                                             user.getName().startsWith("zhu")).collect(Collectors.toSet());
  ```

  