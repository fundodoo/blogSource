---
title: 设计模式-创建型
categories:
  - Java
top: false
date: 2020-03-28 17:04:33
tags:
- java
- 设计模式
keywords: 单例模式,构建者模式,原型模式
description: 
---

## 单例模式
> 在应用的生命周期里，只有一个实例
### 创建方式
#### 双重检测
  ```java
  public class DoubleDetectionSingleton{
    private volatile static DoubleDetectionSingleton singleton;
  
    private DoubleDetectionSingleton(){
  
    }
  
    public static void getInstance(){
      if(null == singleton){
        synchronized(this){
          if(null == singleton){
            singleton = new DoubleDetectionSingleton();
          }
        }
      }
      return singleton;
    }
  }
  ```

  ##### 答疑环节

  - 为什么要判断两次`null`？
    1. 假设两条线程`t1`和`t2`同时执行到第一个`null == singleton`，`t1`继续执行且获取了`锁`,`t2`停在`synchronized(this)`等待锁
    2. `t1`执行结束，释放`锁`,此时`t2`获取`锁`继续执行，此时`singleton == null` 等于`false`，若此时没有判断，则`singleton`将会重新`new`一个新的
  - 案例的写法是否可以被`破坏`？如果可以被破坏，应该怎么防范呢？
    1. 可以。
    2. 单例类增加如下方法`防止`单例被破坏
        ```java
        private void readResolve(){
          return singleton;
        }
        ```

#### 静态内部类
  ```java
  public class InnoClassSingleton{
  
    private InnoClassSingleton(){
  
    }

    private static Class InnoClass(){
      private static final InnoClassSingleton singleton = new InnoClassSingleton();
      System.out.println("测试延迟加载");
    }
  
    public static void getInstance(){
      return InnoClass.singleton;
    }
  }
  ```

  ##### 答疑环节

  - 如何保证单例的？
  - 如何实现延迟加载的？
  - 单例是否会被破坏？

#### 枚举
  ````java
  public class EnumSingleton{
     
     private EnumSingleton(){

     }

     static enum SingletonEnum{
        //创建一个枚举对象，该对象天生为单例
        INSTANCE;

        private EnumSingleton enumSingleton;

        //私有化枚举的构造函数
        private SingletonEnum(){
            enumSingleton = new EnumSingleton();
        }

        public EnumSingleton getInstnce(){
            return enumSingleton;
        }
      }

      //对外暴露一个获取EnumSingleton对象的静态方法
      public static EnumSingleton getInstance(){
          return SingletonEnum.INSTANCE.getInstnce();
      }

  }
  ````

## 原型模式

> 原型模式虽然是创建型的模式，但是与工厂模式没有关系，从名字即可看出，该模式的思想就是将一个对象作为原型，对其进行复制、克隆，产生一个和原对象类似的新对象。

### 优点
1. 根据客户端要求实现动态创建对象，客户端不需要知道对象的创建细节，便于代码的维护和扩展
2. 使用原型模式创建对象比直接new一个对象在性能上要好的多，因为Object类的clone方法是一个本地方法，它直接操作内存中的二进制流，特别是复制大对象时，性能的差别非常明显。所以在需要重复地创建相似对象时可以考虑使用原型模式。比如需要在一个循环体内创建对象，假如对象创建过程比较复杂或者循环次数很多的话，使用原型模式不但可以简化创建过程，而且可以使系统的整体性能提高很多。

### 创建方式
#### 浅复制
- 将一个对象复制后，**基本数据**类型的变量都会**重新创建**，而**引用类型**，指向的还是**原对象**所指向的。

  ```java
  public class Prototype implements Cloneable {
    public Object clone() throws CloneNotSupportedException { 
      Prototype proto = (Prototype) super.clone(); 
      return proto; 
    }
  }
  ```
  > 一个原型类，只需要实现**Cloneable**接口，覆写clone方法，此处clone方法可以改成任意的名称，因为 **`Cloneable`** 接口是个空接口,此处的重点是 **`super.clone()`** 这句话，`super.clone()`调用的是`Object`的clone()方法,而在Object类中，clone()是 **`native`** 的方法

#### 深复制
- 将一个对象复制后，不论是**基本数据**类型还是**引用类型**，都是重新创建的。简单来说，就是**深复制**进行了完全**彻底**的复制，而`浅复制`**不彻底**。

  ```java
  public class Prototype implements Cloneable, Serializable { 
    private static final long serialVersionUID = 1L; 
    private String string; 
    private SerializableObject obj; 

    /* 浅复制 */ 
    public Object clone() throws CloneNotSupportedException { 
      Prototype proto = (Prototype) super.clone(); 
      return proto; 
    }
    
    /* 深复制 */ 
    public Object deepClone() throws IOException, ClassNotFoundException {
      /* 写入当前对象的二进制流 */ 
      ByteArrayOutputStream bos = new ByteArrayOutputStream(); 
      ObjectOutputStream oos = new ObjectOutputStream(bos); 
      oos.writeObject(this); 

      /* 读出二进制流产生的新对象 */ 
      ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray()); 
      ObjectInputStream ois = new ObjectInputStream(bis); 
      return ois.readObject(); 
    }
    
    public String getString() { 
      return string; 
    }
    
    public void setString(String string) { 
      this.string = string; 
    }
    
    public SerializableObject getObj() { 
      return obj; 
    }
    
    public void setObj(SerializableObject obj) { 
      this.obj = obj;
    }
  } 

  class SerializableObject implements Serializable { 
    private static final long serialVersionUID = 1L; 
  }
  ```

### 注意事项
1. 使用原型模式复制对象 **不会调用** 类的`构造方法`。
    - 因为对象的复制是通过调用Object类的clone方法来完成的，它直接在内存中复制数据，因此不会调用到类的构造方法。
    - 构造方法中的代码不会执行，甚至连访问权限都对原型模式无效。
    - 还记得单例模式吗？单例模式中，只要将构造方法的访问权限设置为private型，就可以实现单例。但是clone方法直接 **无视** `构造方法`的权限，所以，单例模式与原型模式是冲突的。
2. 在使用时要注意 **深拷贝** 与 **浅拷贝** 的问题。
    - clone方法**只会**拷贝对象中的`基本`的 **数据类型**，对于 *数组*、*容器对象*、*引用对象*等都**不会**拷贝，这就是`浅拷贝`
    - 要实现深拷贝，必须将原型模式中的数组、容器对象、引用对象等 **另行拷贝**

## 构建者模式
> 将一个复杂对象的构造与它的表示分离，使同样的构建过程可以创建不同的表示，这样的设计模式被称为建造者模式。

### 构造者模式有四个角色：
- builder：为创建一个产品对象的各个部件指定抽象接口
- ConcreateBuilder：实现Builder的接口以构造和装配该产品的各个部件，定义并明确它所创建的表示，并提供一个检索产品的接口
- Director：构造一个使用Builder接口的对象
- Product：表示被构造的复杂对象。

### 构造者模式和工厂模式的区别
- 构造者模式是一种个性化产品的创建；
- 工厂模式是一种标准化的产品创建。

#### 示例
```java
Mybatis -> MappedStatement.java
```


## 工厂模式
### 简单工厂模式
> 工厂类拥有一个工厂方法，接受了一个 **参数** ，通过不同的参数实例化不同的产品类

#### 优缺点
- 优点：简单工厂的特点就是简单粗暴，通过一个含参的工厂方法，我们可以实例化任何产品类
- 缺点：
    1. 任何东西的子类都可以被生产，负担太重。当所要生产产品种类非常多时，工厂方法的代码量可能很庞大。
    2. 在遵循开闭原则（对扩展开放，对修改关闭）的条件下，简单工厂对于增加新的产品无能为力。因为增加新产品只能通过修改工厂方法来实现。

#### 示例
```java
// 简单工厂设计模式（负担太重，不符合开闭原则）
public class AnimalFactory{
  public static Animal createAnimal(String name){
    if("cat".equals(name)){
      return new Cat();
    }else if("dog".equals(name)){
      return new Dog();
    }else if("cow".equals(name)){
      return new Cow();
    }else{
      return null;
    }
  }
}

//静态方法工厂
public class AnimalFactory{

  public static Dog createDog(){
    return new Dog();
  }

  public static Cat createCat(){
    return new Cat();
  }
}
```

### 工厂方法模式
> 工厂方法是针对每一种产品提供一个工厂类，通过不同的工厂实例来创建不同的产品实例。

#### 优缺点
- 优点：工厂方法很好的解决了简单工厂负担太重的问题，因为其将生产每一种产品都分配到不同的产品工厂里；同时新增产品不需要修改原来的类，只需要新增新的工厂即可，使得工厂类符合 开闭原则。
- 缺点：处理产品族饿情况比较复杂。

#### 示例
```java
//抽象出来的工厂对象
public abstract class AnimalFactory{
  //工厂方法
  public abstract Animal createAnimal();
}

//具体的工厂对象1
public class CatFactory extends AnimalFactory{
  @Override
  public Animal createAnimal(){
    return new Cat();
  }
}

//具体的工厂对象2
public class DogFactory extends AnimalFactory{
  @Override
  public Animal createAnimal(){
    return new Dog();
  }
}

```

### 抽象工厂模式
> 抽象工厂是应对产品族概念的。

- 工厂方法生产一个系列的产品。抽象工厂不止一个 ***方法***。
