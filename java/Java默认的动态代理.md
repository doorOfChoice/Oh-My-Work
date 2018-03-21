# Java默认的动态代理

## 0x00 静态代理

在使用动态代理前，有一种代理的方式叫做``静态代理``, 假设如下A实现了Play接口

```java
interface Play {
  void play();
}

class A implements Play{
  public void play() {
    //play的动作
  }
}
```

现在如果我想在A执行play之前做一些额外的动作，可以通过如下暴力方式

```java
class B implements Play {
  private A a;
  public B(A a){this.a = a;}
  public void play(){
    //do before
    a.play();
  }
}
```

这种手动在需要代理的位置进行添加额外动作的方式称之为静态代理



## 0x01 动态代理

当然，java里面有更聪明的方式，即动态代理

java 默认的动态代理依赖于接口`` InvocationHandler``, 该接口有如下方法

```java
//proxy: 代理的实例对象
//method: 调用的方法
//args: 调用方法传递的参数
//返回值：动态调用method之后要返回的值，一般返回method调用后原始返回的值
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable ;
```

以下是一个简单的实现

```java
class AProxy implements InvocationHandler {
  private A a;
  
  public AProxy(A a){this.a = a;}
  
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    System.out.println("invoke before");
    Object result = method.invoke(a, args);
    System.out.println("invoke after");
    return result;
  }
}
public class Main {
	public static void main(String[] args) {
      	A a = new A();
	    Play play = (PLay)Proxy.newProxyInstance(
          Main.class.getClassLoader(), //类加载器
          a.getClass().getInterfaces(), //代理的接口
          new AProxy(a) //实现了InvocationHandler的对象
        );
      	play.play();
	}
}

```

这串代码会在调用play()方法前后分别打印出"invoke before" 和 "invoke after", 而不用再去新建一个类。

不过java默认的动态代理只能通过接口代理，而不能通过类，更不能通过类抽象出接口，这是比较坑的。



## 0x02 invoke中proxy参数是什么

官方给的话很简单，十分的简单

``the proxy instance that the method was invoked on``

也就是说一个代理的实例对象，但是这个代理的实例对象又是什么呢？？？

我们通过一反编译proxy对象，看看其内部结构。

```java
//生成代理类的字节码
byte[] buf = ProxyGenerator.generateProxyClass(proxy.getClass().getSimpleName(), proxy.getClass().getInterfaces());
FileOutputStream fo = new FileOutputStream(proxy.getClass().getSimpleName() + ".class");
fo.write(buf);
fo.flush();
fo.close();
```

调用上面这串代码后，生成了如下的类

```java
public final class $Proxy0 extends Proxy implements Play {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
      	//把var1传给父类构造器，super.h就是对应的var1
        super(var1);
    }
	//这是调用Object类的equals方法
    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }
	//这是调用Object类的toString方法
    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
	//这是调用Play接口的play方法

    public final String play() throws  {
        try {
            return (String)super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
	//这是调用Object类的hashCode方法
    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("Play").getMethod("play");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}

```

从上面的反编译代码可以分析出来

1. 其实java默认的动态代理就是动态的代替静态代理手工添加代码的行为。那么invoke的proxy参数也就是这个$Proxy类，之所以``Proxy.newProxyInstance``能够转换成对应的接口，也是因为proxy实现了传递进去的接口.
2. 其实每个invoke的调用也都是$Proxy调用invoke接口，这也就是为什么调用proxy的方法会导致死循环的原因了。

## 0x03 缺点

我能感受到的缺点就是

1. 不能够代理类
2. 不能从类抽象出接口