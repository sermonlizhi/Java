### 一、常用集合继承树结构

- Collection接口

<img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210514102843922.png" alt="image-20210514102843922" style="zoom:50%;" />



- Map接口

  <img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210514104612152.png" alt="image-20210514104612152" style="zoom:50%;" />

### 二、Iterable和Iterator

- Iterable是所有集合(List、Set、Queue)的顶级接口，在JDK1.5以后，引入了Iterable，使用foreach语句(增强for循环)必须使用Iterable；其继承结构如下

  <img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210514094852759.png" alt="image-20210514094852759" style="zoom: 50%;" />

- Iterator定义了迭代逻辑的对象，Iterable接口提供了iterator()方法，返回一个Iterator对象，通过Iterator对象对集合进行遍历，Iterator主要的方法如下

  <img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210514095302026.png" alt="image-20210514095302026" style="zoom:75%;" />

  

### 



### 