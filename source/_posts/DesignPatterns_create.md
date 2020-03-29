---
title: 设计模式-创建型
categories:
  - Java
top: false
date: 2020-03-28 17:04:33
tags:
- java
- 设计模式
keywords:
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
## 构建者模式
## 工厂模式
