---
title: Java_基础知识
categories:
  - Java
top: false
date: 2020-03-30 14:01:48
tags:
- Java
keywords:
description: 
---

## Java基础知识

### 重载和重写的区别
- **重载** ： 发生在`一个类`里。`方法名`必须 **相同** ；方法返回值和访问修饰符可以不同；发生在`编译期`。以下情况满足任意一个即可。
    - `参数类型` **不同**；
    - `参数个数` **不同**；
    - `参数顺序` **不同**；
- **重写** ： 发生在`父子类`中， **方法名**、 **参数** 列表必须 **相同** ；`返回值`范围 **小于等于** 父类；抛出的`异常`范围 **小于等于** 父类；`访问修饰符`范围 **大于等于** `父类`；如果父类的访问修饰符为`private`则子类 **不能** 重写；如果父类的方法有`final`修饰，子类也 **不能** 重写；

### String和StringBuffer、StringBuilder的区别？String为什么是不可变的？
- **String** : 不可修改的，在内部是`private final char value[]` 的数据结构存储数据。
- **StringBuffer** : 继承`AbstractStringBuilder`, `char[] value`结构存储；方法使用了`synchronized`修饰，是线程安全的。
- **StringBuilder** : 继承`AbstractStringBuilder`, `char[] value`结构存储；方法未使用锁，是非线程安全的。

#### 三者使用建议
- `String` 适合操作少
- `StringBuilder` **单线程** *大量* 操作字符串缓冲区
- `StringBuffer` **多线程** *大量* 操作字符串缓冲区

### 自动装箱与拆箱
- **装箱** ：将基础类型用它们对应的引用类型包装起来。eg： `int -> Integer`
- **拆箱** ：将包装类型转换为基础的数据类型；   eg  `Long -> long`

### == 与 eauals() 的区别
- `==` : 对于 ***基础类型*** `==` 比较的是 ***值*** 是否相等；而 ***引用类型*** `==` 比较的是 ***内存地址***。
- `equals()` : 判断两个对象是否相等。
    1. 类 **没有** ***重写*** `equals()`方法。则通过 `equals()` 比较该类的两个对象时， ***等价于*** `==`；
    2. 类 ***重写** 了 `equals()`，一般在重写equals()时也会重写`hashCode()`方法，则是通过`equals()`里的规则判断两个对象是否相等。

### final关键字的总结
- 修饰 **变量**： **基础类型** -> 初始化后 ***不可*** 更改； **引用类型** -> 初始化后 ***不能*** 指向一个新对象。
- 修饰 **方法**： 该方法 ***不能*** 被 **重写**。
- 修饰 **类**：则这个类 ***不能*** 被 `继承`，`final类`中的所有成员、方法都会隐式的被final修饰。

### Object类的常见方法

### Java中的异常处理

### Java获取键盘输入的常用方法
- 通过Scanner
  ```java
  Scanner input = new Scanner(System.in);
  String s = input.nextLine();
  input.close();
  ```
- 通过BufferedReader
  ```java
  BufferedReader input = new BufferedREader(new InputStreamReader(System.in));
  String s = input.readLine();
  ```

### 接口和抽象类的区别
- **接口** 是对`对象行为`的抽象，是对对象`行为`的规范；**抽象**是对类的抽象，是对类的`metaData`的描述，类似于 *模版* 定义。
- 一个类可以实现 `多个` ***接口***；但是一个类只能继承`一个` ***抽象类***
- 实现`接口`就必须 *实现* 接口定义的 ***所有*** 方法(JDK &lt; 1.8),而继承`抽象类`就只需要实现 ***抽象方法*** 就可以，也 *不建议* 重写非抽象方法。
- **接口** 的方法默认都是`public`。

#### JDK8接口说明
- JDK8接口可以定义`default`实现
    - 实现类可以不用实现`default`方法
    - 若一个类实现了`两个接口`，且两个接口都定义了 ***相同*** 名称的 ***默认方法***，则实现类就 ***必须*** `重写`该名称的方法。否则会 ***报错***。
- JDK8接口可以定义static方法，只能使用接口调用。
  ```java
  public interface TestJdk8Interface{
    void test(); //普通接口方法

    /**
    *拥有默认实现的接口方法
    */
    default void testDefault(){
      System.out.println("default");
    }

    /**
    *接口的默认静态方法
    */
    static void testStatic(){
      System.out.println("static");
    }

  }

  public class TestJdk8InterfaceImpl implements TestJdk8Interface{
    @Override
    public void test(){
      System.out.println("common");
    }

    public static void main(String[] args){
      TestJdk8Interface.testStatic();//使用接口的静态方法
      TestJdk8Interface f = new TestJdk8InterfaceImpl();
      f.test();
      f.testDefault(); //调用有默认实现的接口方法
    }
  }
  ```

