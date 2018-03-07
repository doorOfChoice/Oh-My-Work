# Effective java学习

面试之前草草的看了一下《effect java》

* 创建和销毁对象

## 创建和销毁对象

### 考虑用静态工厂方法代替构造器

1. 静态工厂方法的具有特定的名称，方便知道创建目的
2. 静态工厂方法可以实现单例模式，比如Integer.valueOf(x), 如果-127<=x<=128，就始终返回一个对象
3. 静态工厂方法可以返回类型的任何子类型
4. 可以使创建变得更加简洁

### 遇到多个构造器参数时考虑用构建器

​	如果有这样一个类

```java
class Peron {
  private String name;
  private int age;
  //.....get and set方法省略
}
```

​	可以采用如下方式进行构建

```java
class Peron {
  private String name;
  private int age;
  static class Builder {
    private String name;
    private int age;
    public Person build(){
      Person p = new Person();
      p.name = name;
      p.age = age;
      return p;
    }
    public void name(String name){this.name = name;}
    public void age(int age){this.age = age;}
  }
  public static Builder builder(){
    return new Builder();
  }
}

public class Test{
  public static void main(String[] args) {
    Person p = Person.builder().name("WangRui").age(10).build();
  }
}
```

采用构建器构建，当在构造器或者静态工厂参数很多的时候，更加简洁，可视。但是因为每次创建类都会构建一个构建器，相对于普通创建会有一定的性能损失。需要自己依照自己的场景选择要或者不要使用构建器。

### 用私有构造器或者枚举类型强化Singleton属性

在Spring容器管理中，会经常使用到单例，使用单例可以减少内存中的对象数量。使用单例的前提是，此对象会经常被重复使用。

```java
//私有静态final对象
class Elivis {
  private static final Elivis INSTANCE = new Elivis();
  public static Elivis getInstance() {return INSTANCE;}
}
//枚举
enum Elivis {
  INSTANCE;
}
```

### 通过私有构造器强化不可实例化的能力

### 避免创建不必要的对象

1. 像``String s = "hello world"``, 不要写成``String s = new String("hello world")``, 字面量的字符串是有缓存的，能够重复使用。
2. 能用基本类型，就不用装箱基本类型, 比如以下代码就有巨大的性能损失，因为进行了大量的自动装箱操作

```java
Long sum = 1L;
for(long i = 0; i < 1000000; i++)
  sum += i;
```

3. 当你能够重用当前对象的时候就尽量不创建对象

### 消除过期的对象引用

以下是一个简单的Stack代码

```java
class Stack {
  private Object[] elements;
  private int size = 0;
  private static final int DEFAULT_INITAIL_CAPACITY = 16;
  public Stack(){
    elements = new Object[DEFAULT_INITAIL_CAPACITY];
  }
  public void push(Object e) {
    ensureCapacity();
    elements[size++] = e;
  }
  public Object pop() {
    if(size == 0)
      throw new EmptyStackException();
    return elements[--size];
  }
  private void ensureCapacity() {
    if(elements.length == size)
      elements = Arrays.copyOf(elements, 2 * size + 1);
  }
}
```

当往这个Stack里面加入很多元素之后，然后删除元素的时候，只是减少了size的值，而没有真正的释放被变量引用的对象，那么垃圾回收期将不会回收这些对象。正确的做法应该是如下：

```java
 public Object pop() {
    if(size == 0)
      throw new EmptyStackException();
    elements[size] = null;
    return elements[--size];
  }
```

### 避免使用终结方法

终极方法类似于C++中的析构函数，但是又不尽相同。

```java
protected void finalize() throw Throwable{}
```

终结方法会在垃圾回收对象之前调用，也就意味着，即使一个对象变得不可达了，也要等到垃圾回收器回收到该对象得时候才能执行该方法，况且终结方法再jvm是保持低优先级得，不一定立即调用。

缺点

1. finalize可能导致对象重生
2. finalize至多由gc执行一次
3. finalize可能带来性能问题，因为jvm通常再单独的低优先级栈中完成执行