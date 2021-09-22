## 一、Optional简介与使用

#### 1.1 Optional简介

在Java应用开发的过程中，NPE(NullPointerException)问题是一个非常常见的问题，为了避免空指针异常对代码的破坏，我们不得不在各种可能出现空指针的地方通过if-else来校验，造成代码累赘，严重影响代码可读性。为了优雅的解决NPE问题，Google公司著名的Guava项目引入了Optional类，Guava通过检查空值的方式来防止代码污染，它鼓励程序员编写简单干净的代码。受到Google Guava项目的启发，Optional类已经成为Java 8类库的一部分

Optional<T>类(java.util.Optional)是一个容器类，它可以保存类型T(任意类型)的值，成员变量value代表了这个值，或者仅保存null，表示这个值不存在，原来用null来表示一个值不存在，现在Optional可以更好的表达这个概念，并且可以避免空指针异常。通过使用Optional可以减少代码中的判空，实现函数式编程。

#### 1.2 基本使用

```java
//常见三个实体类,使用lombok自动生成相关getter和setter以及toString方法,相关注解省略
public class Member {
    private int id;
    private String name;
    private int age;
    private Address address;
}

public class Address {
    private String detailAdd;
    private Country country;
}

public class Country {
    private String countryName;
    private String countryCode;
}
```

**获取用户所在地方的编号**

```java
//通过if-else来实现,需要进行层层判空检验
public String getMemberCountryCode(Member member){
    String countryCode = "";
    if (member != null){
        Address address = member.getAddress();
        if (address != null){
            Country country = address.getCountry();
            if (country != null){
                countryCode = country.getCountryCode();
            }
        }
    }
    return countryCode;
}
```

```java
//通过Optional类来实现
public String getMemberCountryCode(Member member){
    String countryCode = "";
    Optional<String> countryCodeOptional = Optional.ofNullable(member).map(Member::getAddress).map(Address::getCountry).map(Country::getCountryCode);
    if (countryCodeOptional.isPresent()){
        countryCode = countryCodeOptional.get();
    }
    return countryCode;
}
```

`通过上面的例子,可以明显的看出,使用Optional类可以明显减少不必要的冗余代码,只需要在最后做一次判空检验即可,为空的时候还可以使用Optional的orElse、orElseGet来指定默认值,或者通过orElseThrow来抛出异常`

```java
//为空时,返回指定的默认值
countryCode = Optional.ofNullable(member).map(Member::getAddress).map(Address::getCountry).map(Country::getCountryCode).orElse("0710");
```

当Optional为空时，调用它的toString()方法也不会导致空指针，而是返回字符串"Optional.empty"

```java
Member member = null;
Optional<Member> memberOptional = Optional.ofNullable(member);
System.out.println(memberOptional);

//输出结果
Optional.empty
```

#### 1.3 常用方法

- Optional对象的创建

  在使用Optional时首先需要创建Optional对象，但Optional类的两个构造方法都是private型的，可通过Optional类提供的三个静态方法来创建Optional对象

  ```java
  //创建包装对象值为null的Optional对象,一般不使用
  Optional<Object> emptyOptional = Optional.empty();
  
  //创建包装对象不为空的Optional对象
  Optional<String> optional = Optional.of("lizhi");
  
  //创建包装对象可以为null的Optional对象
  Optional<String> nullAbleOptional = Optional.ofNullable(null);
  ```

- 数据处理

  Optional提供了map、filter以及flatMap方法来对包装对象进行操作

  ```java
  //通过map映射获取member对象的Address的包装Optional对象
  Optional<Address> addressOptional =  Optional.ofNullable(member).map(Member::getAddress)
  ```

  `如果member为空,则会返回一个空的Optional对象,不会导致空指针`

  ```java
  //通过filter判断member对象年龄是否大于24
  Optional<Member> memberOptional = Optional.ofNullable(member).filter(m -> m.getAge() > 24);
  ```

  `filter方法中有值且满足判断条件则返回包含该值的Optional,否则返回空的Optional`

  ```java
  //使用flatMap方法获取映射的address的包装Optional对象
  Optional<Address> addressOptional = Optional.ofNullable(member).flatMap(m -> Optional.ofNullable(m.getAddress()));
  ```

  `flatMap方法与map方法基本一样，不同的是map方法的mapping函数返回的可以是任何类型T,flatMap方法的mapping函数返回的必须是Optional类型`

- 获取值

  Optional类主要提供了四种获取值的方法，但这些方法又各不相同

  ```java
  //get()方法,如果Optional为空,则get方法将抛出异常
  Member memberInfo = Optional.ofNullable(member).get();
  
  //orElse()方法,如果Optional不为空,则将其返回,如果为空则返回指定的值
  Member memberInfo = Optional.ofNullable(member).orElse(new Member(1, "lizhi", 24, null));
  
  //orElseGet()方法,与orElse()方法类似,区别在于指定默认值的方式不同,orElse直接将传入的值作为默认值,而orElseGet()则通过
  Member memberInfo = Optional.ofNullable(member).orElseGet(() -> new Member(1, "lizhi", 24, null));
  
  //orElseThrow()方法,如果Optional不为空则将其返回,否则直接抛出异常
  Member memberInfo = Optional.ofNullable(member).orElseThrow(IllegalArgumentException::new);
  ```

## 二、Optional源码分析

#### 2.1 构造方法

```java
//空的Optional实例
private static final Optional<?> EMPTY = new Optional<>();
//Optional包装的原始数据,可以是任何类型,用变量value表示
private final T value;

//无参构造
private Optional() {
    this.value = null;
}

//有参构造
private Optional(T value) {
    this.value = Objects.requireNonNull(value);
}
```

`Optional类提供了两个私有的构造方法,仅供类内部调用`

#### 2.2 实例方法

```java
//直接返回Optional类的空实例EMPTY
public static<T> Optional<T> empty() {
    @SuppressWarnings("unchecked")
    Optional<T> t = (Optional<T>) EMPTY;
    return t;
}

//参数不能为null,有参构造器对参数进行了判空检验,如果参数为null,将会导致空指针
public static <T> Optional<T> of(T value) {
    return new Optional<>(value);
}

//参数可以为空,如果参数为空相当于调用了empty方法,参数不为空时相当于调用了of方法
public static <T> Optional<T> ofNullable(T value) {
    return value == null ? empty() : of(value);
}
```

#### 2.3 判断是否为空

```java
public boolean isPresent() {
    return value != null;
}

//如果源数据不为null,则调用参数中的accept方法对数据进行处理;如果为空则什么也不做
public void ifPresent(Consumer<? super T> consumer) {
    if (value != null)
        consumer.accept(value);
}
```

#### 2.4 数据处理

```java
//如果过滤条件为null,则直接抛出空指针异常,如果Optional为空则直接返回空实例,否则判断是否满足过滤条件,如果满足则返回实例,不满足就返回一个空的Optional实例
public Optional<T> filter(Predicate<? super T> predicate) {
    Objects.requireNonNull(predicate);
    if (!isPresent())
        return this;
    else
        return predicate.test(value) ? this : empty();
}

//与filter的结构一致,先判断参数是否为空,然后判断当前Optional实例时是否为空,不为空则执行apply映射函数,然后将映射后的数据封装成Optional类的实例返回
public<U> Optional<U> map(Function<? super T, ? extends U> mapper) {
    Objects.requireNonNull(mapper);
    if (!isPresent())
        return empty();
    else {
        return Optional.ofNullable(mapper.apply(value));
    }
}

//与map逻辑基本一致,map方法的apply函数返回的是任意类型,而flatMap方法的apply函数返回的只能是Optional类型或者null,所以flatMap对返回值进行了判空检验,如果映射函数返回的是null则直接抛出空指针异常
public<U> Optional<U> flatMap(Function<? super T, Optional<U>> mapper) {
    Objects.requireNonNull(mapper);
    if (!isPresent())
        return empty();
    else {
        return Objects.requireNonNull(mapper.apply(value));
    }
}
```

#### 2.5 数据获取

```java
//get()方法,如果Optional的实例为空时,则抛出以下异常信息
public T get() {
    if (value == null) {
        throw new NoSuchElementException("No value present");
    }
    return value;
}

//指定与原始类型相同的默认值,如果Optional不为空则返回该实例包装的数据,否则返回默认值
public T orElse(T other) {
    return value != null ? value : other;
}

//与orElse()方法一致,不同的是orElseGet()方法可以通过函数式接口来指定默认值
public T orElseGet(Supplier<? extends T> other) {
    return value != null ? value : other.get();
}

//如果Optional实例不为空,则返回原始包装数据,如果为空就抛出定义的异常
public <X extends Throwable> T orElseThrow(Supplier<? extends X> exceptionSupplier) throws X {
    if (value != null) {
        return value;
    } else {
        throw exceptionSupplier.get();
    }
}
```

#### 2.6 Object方法

```java
//即使Optional为一个空实例,调用toString()方法也不会出现空指针
public String toString() {
    return value != null
        ? String.format("Optional[%s]", value)
        : "Optional.empty";
}
```

