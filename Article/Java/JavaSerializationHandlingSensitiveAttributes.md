<h2> Java, Serialization处理敏感属性 </h2>

- [父类的序列化与Transient关键字](#%E7%88%B6%E7%B1%BB%E7%9A%84%E5%BA%8F%E5%88%97%E5%8C%96%E4%B8%8Etransient%E5%85%B3%E9%94%AE%E5%AD%97)
- [序列化类的私有方法说明](#%E5%BA%8F%E5%88%97%E5%8C%96%E7%B1%BB%E7%9A%84%E7%A7%81%E6%9C%89%E6%96%B9%E6%B3%95%E8%AF%B4%E6%98%8E)
  - [对敏感属性加密](#%E5%AF%B9%E6%95%8F%E6%84%9F%E5%B1%9E%E6%80%A7%E5%8A%A0%E5%AF%86)
  - [清除敏感属性](#%E6%B8%85%E9%99%A4%E6%95%8F%E6%84%9F%E5%B1%9E%E6%80%A7)

> 对于**处理敏感属性**,本文整理了三种方式:
> 1) 父类的序列化与Transient关键字.<br/>
> 2) 序列化类的私有方法说明.<br/>
> 2.1) 对敏感属性加密(使用序列化类的私有方法).<br/>
> 2.2) 清除敏感属性(使用序列化类的私有方法).<br/>

``` ps. 没有最好的方式,只有最适合的方式 ```
<hr>

### 父类的序列化与Transient关键字

```java
// 序列化父类(未在无参构造函数中初始化变量)
public class A {
    private String a;
    private String b;
    private String c;
    // set/get 略
}
```
```java
// 序列化类
public class B implements Serializable extends A {
    private static final long serialVersionUID = 1L;
    private String d;
    private transient String e;
    // set/get 略
}
```
> **transient关键字的作用是控制变量的序列化,在变量声明前加上该关键字,可以阻止该变量被序列化.**<br/>
> **同样,在被反序列化后,transient变量的值被设为初始值,如int型的是'0',对象型的是'null'.**<br/>

> **父类若想序列化,就需要让父类实现Serializable接口.若父类不实现的话,就需要有默认的无参的构造函数.**
> 1. 在父类没有实现Serializable接口时,虚拟机是不会序列化父对象的,<br/>
> 而Java对象的构造必须先有父对象,才有子对象,反序列化也同样.<br/>
> 2. 在反序列化时,为了构造父对象,会调用父类的无参构造函数作为默认的父对象.<br/>
> 因此当我们取父对象的变量值时,它的值是调用父类无参构造函数后的值.<br/>
>
> **ps.**<br/>
> 如果你考虑到这种序列化的情况,可在父类'无参构造函数'中对变量进行初始化.<br/>
> 否则,父类变量值都是默认声明的值.如int型的默认是'0',String型的默认是'null'.<br/>
<hr>

### 序列化类的私有方法说明

> 序列化有个机制叫'**序列化回调**',当把一个'对象'转换为'流数据'时,会通过反射检查序列化类是否有以下两个方法.<br/>
> 即在序列化过程中，JVM会试图调用对象类里的**writeObject**和**readObject**方法,<br/>
> 若其方法满足'为私有化'、'无返回值'等特性,则会使用该方法进行序列化和反序列化.<br/>
> 若没有，则默认调用是**ObjectOutputStream**的**defaultWriteObject**和**ObjectInputStream**的**defaultReadObject**方法继续序列化.<br/>

#### 对敏感属性加密

> 使用序列化类的私有方法<br/>

```java
public class A implements Serializable {
    private static final long serialVersionUID = 1L;
    private String password = "password";
    // set/get方法略
    private void writeObject(ObjectOutputStream out) {
        try {
            ObjectOutputStream.PutField putFields = out.putFields();
            password = "encryption"; // 加密
            putFields.put("password", password);
            out.writeFields();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    private void readObject(ObjectInputStream in) {
        try {
            ObjectInputStream.GetField readFields = in.readFields();
            Object object = readFields.get("password", "");
            password = "password";//模拟解密,需要获得本地的密钥
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
```java
public static void main(String[] args) {
    try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("/tmp/tmp.bin"))) {
        out.writeObject(new A());
    } catch (Exception e) {
        e.printStackTrace();
    }
    try (ObjectInputStream oin = new ObjectInputStream(new FileInputStream("/tmp/tmp.bin"))) {
        A t = (A) oin.readObject();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

#### 清除敏感属性

> 使用序列化类的私有方法<br/>
> 不同于加密,按照以下逻辑,A类是部分属性传输(非传输部分直接给予'0'或'null')<br/>

```java
public class A {
    private int aa;
    private int bb;
    // set/get、构造方法略
}
```
```java
public class B implements Serializable {
    private static final long serialVersionUID = 1L;
    private int account;
    private transient A a;
    // set/get、构造方法略
    private void writeObject(java.io.ObjectOutputStream out) throws Exception {
        out.defaultWriteObject();
        out.writeBytes(a.getAa());
    }
    private void readObject(java.io.ObjectInputStream in) throws Exception {
        in.defaultReadObject();
        a = new A(in.readInt(), 0);  // 直接将bb赋值为0
    }
}
```
<hr>
