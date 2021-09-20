## 一、Lambda表达式

#### Lambda表达式的格式

举例：(o1,o2) -> Integer.compare(o1,o2);

- "->"：lambda操作符或箭头操作符
- "->左边"：lambda形参列表(其实就是接口中的抽象方法的参数列表)
- "->右边"：lambda体(其实就是重写接口的抽象方法的方法体)

`Lambada表达式的本质就是接口(函数式接口)的实例`

#### 1.1 无参写法

```java
Runnable rab1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello World");
    }
};
rab1.run();

/** 无参时"->"左边需要有一个空的括号() */
Runnable rab2 = () -> System.out.println("Hello World");
rab2.run();
```

#### 1.2 一个参数写法

```java
Consumer con1 = new Consumer() {
    @Override
    public void accept(Object o) {
        System.out.println(o);
    }
};
con1.accept("Hello World");

/** 一个参数时，可以忽略参数列表的括号() */
Consumer con2 = string -> System.out.println(string);
con2.accept("Hello World");
```

#### 1.3 多个参数

```java
Comparator<Integer> com1 = new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        return Integer.compare(o1,o2);
    }
};
int compare1 = com1.compare(11,22);
System.out.println(compare1);

/** 多个参数时,参数需要用括号()包裹*/
Comparator<Integer> com2 = (o1,o2) -> Integer.compare(o1,o2);
int compare2 = com2.compare(11,22);
System.out.println(compare2);
```

#### 1.4 方法体包含多条执行语句

```java
Comparator<Integer> com1 = new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        System.out.println(o1);
        System.out.println(o2);
        return Integer.compare(o1,o2);
    }
};
int compare1 = com1.compare(11,22);
System.out.println(compare1);

/** 当方法体包含多条语句时,需要用{}包裹 */
Comparator<Integer> com2 = (o1,o2) -> {
    System.out.println(o1);
    System.out.println(o2);
    return Integer.compare(o1,o2);
};
int compare2 = com2.compare(11,22);
System.out.println(compare2);
```

`如果Lambda体只有一条执行语句,则不需要"{}"和return,如果有多条执行语句时,"{}"和return必须都要有`

## 二、函数接口

#### 2.1 函数接口定义

只包含一个自定义抽象方法的接口称为函数式接口，不论有没有@FunctionalInterface注解，它都是一个函数式接口，使用注解可以检查它是否是一个函数式接口，同时javadoc也会包含一个声明，说明这个接口是一个函数式接口

@FunctionalInterface注解的接口中，必须满足只有一个非static和default的函数才能称之为函数接口，不然注解会报错，但接口中可以存在Object类的方法

```java
@FunctionalInterface
public interface Test {
    void test(String str);

    @Override
    boolean equals(Object object);

    @Override
    String toString();

    static void print(){
        System.out.println("");
    }

    default void show(){
        System.out.println("");
    }
}
```

`上面的代码展示了一个函数式接口,可以重写Object类的public方法,可以有static和default方法,但只能有一个自定义的抽象方法`

#### 2.2 函数式接口的使用

- 定义一个函数式接口，有且仅有一个自定义抽象方法，但可以有Object类的public方法，protected方法不行

  <img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210525170444658.png" alt="image-20210525170444658" style="zoom:75%;" />

- 方法中调用函数接口的方法

  <img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210514151650310.png" alt="image-20210514151650310" style="zoom:75%;" />

- 外部方法调用该类的方法时，可以自定义实现函数接口的函数，格式(函数接口方法参数)->{具体实现}

  <img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210514151718165.png" alt="image-20210514151718165" style="zoom:75%;" />

#### 2.3 内置四大函数式接口

- Consumer<T>： 消费型接口

  ```java
  void accept(T t);
  ```

- Supplier<T>：供给型接口

  ```java
  T get();
  ```

- Function<T,R>：函数型接口

  ```java
  R apply(T t);
  ```

- Predicate<T>：断言型接口

  ```java
  boolean test(T t);
  ```
  
  断言型接口的使用，过滤集合元素
  
  ```java
  public void test(){
      List<String> list = Arrays.asList("lizhi","linan","zhuyuzhu");
      List<String> filterList = filterString(list,name -> name.contains("zh"));
      System.out.println(filterList);
  }
  
  /** 将满足条件的元素装入新的列表返回 */
  private List<String> filterString(List<String> list, Predicate<String> predicate){
      List<String> filterList = new ArrayList<>();
      for (String name: list) {
          if (predicate.test(name)){
              filterList.add(name);
          }
      }
  
      return filterList;
  
  }
  ```

`在大多数场景中,内置的四种函数式接口都可以满足开发需求,在java.util.function包下面还包含了很多基于这四种函数式接口拓展的其他函数式接口`

上面函数式接口使用的例子可以直接用消费型接口实现

```java
Demo demo = new Demo();
demo.method("Hello World",(str)-> {
    System.out.println("this is functional interface");
    System.out.println(str);
});

public void method(String str, Consumer test){
    System.out.println("This is main method");
    test.accept(str);
}
```

`Lambda表达式是基于函数式接口实现的`

## 三、方法引用与构造引用

当要传递给Lambda体的操作，已经有实现的方法，则可以使用方法引用。意思就是说原来Lambda体的操作是执行某个类或对象的方法，这时Lambda体就可以采用方法引用来实现，方法引用其实就是Lambda表达式的语法糖，是一种更高层次的表达而已

要求：实现接口抽象方法的参数列表和返回值类型，必须与方法引用的方法的参数列表和返回值类型保持一致

```java
/** 格式 */
类名/对象 :: 方法;  //通过“::”将类或对象与方法名分开,表达Lambda方法体执行的是该方法
//主要有三种使用情况
对象 :: 实例方法名
  类 :: 静态方法名
  类 :: 实例方法名  //这种情况上面的要求不再符合
```

`调用静态方法只能通过类,不能通过对象来调`

#### 3.1 方法引用

##### 3.1.1 无返回值且参数列表一致

```java
/**
 *	Consumer的方法：void accepte(T t)
 *	System.out表示的是PrintStream
 * 	PrintStream的方法：void println(T t)
 */
//lambda表达式
Consumer con1 = s -> System.out.println(s);
con1.accept("Hello World");
//方法引用
Consumer con2 = System.out :: println;
con2.accept("Hello World");
```

##### 3.1.2 返回值类型一致且无参数列表

```java
/**
 *	Supplier的方法：T get()
 *	User的方法：String getName()
 */
//lambda表达式
User user = new User(1,"lizhi","man","159");
Supplier sup1 = () -> user.getName();
System.out.println(sup1.get());

//方法引用
Supplier sup2 = user :: getName;
System.out.println(sup2.get());
```

##### 3.1.3 有参数列表且返回值类型一致

```java
//方法引用的方法的参数列表和compare抽象方法的参数列表一致
Comparator<Integer> com1 = (o1,o2) -> Integer.compare(o1,o2);
int result1 =  com1.compare(11,22);

Comparator<Integer> com2 = Integer :: compare;
int result2 = com2.compare(11,22);
```

```java
//apply方法和round的方法对应的返回值和参数列表类型一致
Function<Double,Long> function1 = d -> Math.round(d);
long result1 = function1.apply(0.5);

Function<Double,Long> function2 = Math :: round;
long result2 = function2.apply(0.5);
```

##### 3.1.4 类调实例方法

```java
/**
 *	Comparator的方法：int compare(T t1,T t2)
 *	String的方法：int compareTo(T t)  由实例发起调用
 */
Comparator<String> com1 = (c1,c2) -> c1.compareTo(c2);
int result1 = com1.compare("dag","aia");

Comparator<String> com2 = String :: compareTo;
int result2 = com2.compare("dag","aia");
```

这个写法理解起来比较难，Comparator的抽象方法compare包含两个String类型的参数，而调用String的compareTo方法只有一个参数，compare中的第一次参数主要的作用就是调用的发起者，同时第一个参数也是String的实例，所以通过类调用实例方法，其实是因为Comparator抽象方法的第一次参数是该类的实例

```java
/**
 *	BiPredicate的方法：boolean test(T t1,U t2)
 *	String的方法：boolean equals(T t)  由实例发起调用
 */
BiPredicate<String,String> bi1 = (b1,b2) -> b1.equals(b2);
bi1.test("lizhi","linan");

BiPredicate<String,String> bi2 = String :: equals;
bi2.test("lizhi","linan");
```

下面的例子中，apply抽象方法有一个参数，getName方法没有参数，但getName方法需要有一个发起调用的实例，所以apply方法的参数就是充当User实例的角色

```java
/**
 *	Function的方法：R apply(T t)
 *	User的方法：String getName()  由实例发起调用
 */
User user = new User(1,"lizhi","man","159");

Function<User,String> fun1 = u -> u.getName();
String userName1 =  fun1.apply(user);

Function<User,String> fun2 = User :: getName;
String userName2 = fun2.apply(user);
```

#### 3.2 构造引用

##### 3.2.1 无参构造

```java
Supplier<User> sup1 = () -> new User();
User user1 = sup1.get();

Supplier<User> sup2 = User::new;
User user2 = sup2.get();
```

##### 3.2.2 单参数构造

```java
//User有一个参数为int类型的构造器，无需指明参数类型
Function<Integer,User> fun1 = id -> new User(id);
User user1 = fun1.apply(100);

Function<Integer,User> fun2 = User :: new;
User user2 = fun2.apply(200);
```

##### 3.2.3 多参构造器

```java
BiFunction<Integer,String,User> bif1 = (id,name) -> new User(id,name);
User user1 = bif1.apply(100,"lizhi");

BiFunction<Integer,String,User> bif2 = User :: new;
User user2 = bif2.apply(200,"linan");
```

#### 3.3数组引用

```java
Function<Integer,String[]> fun1 = length -> new String[length];
String[] array1 = fun1.apply(5);

Function<Integer,String[]> fun2 = String[] :: new;
String[] array2 = fun2.apply(5);
```

