---
title: 'Effective Java: 构造器相关'
date: 2017-01-18 13:09:46
tags: Java
categories:
 - 读书笔记
 - Effective Java
---

整理自我的Wiz笔记  

---
# 用私有构造器或者枚举类型强化Singleton属性
## Singleton
仅仅被实例化一次的类，通常被用来代表那些本质上唯一的系统组件，比如窗口管理器或者文件系统。  
java1.5之后，有三种方式实现singleton。

### Singleton with public final field
```java
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() {...}
    public void leaveTheBuilding() {...}
}
```
调用一次private 构造器，实例化公有的静态final域Elvis.INSTANCE。  
__特权客户端可以借助AccessibleObject.setAccessible，通过反射机制调用私有构造器！__
<!-- more -->
### Singleton with static factory
```java
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() {...}
    public static Elvis getInstance() {return INSTANCE;}
    
    public void leaveTheBuilding() {...}
}
```
该方法的优势在于，提供了灵活性。  

### Enum Singleton
```java
public enum Elvis {
    INSTANCE;
    public void leaveTheBuilding() {...}
}
```
该方法更加简洁，并提供了序列化机制。__单元素的枚举类型已经成为实现Singleton的最佳方法！__  
## 总结
根据面神的经验,__在公司中为了可读性和其他原因仍然会选择使用静态类来实现，不会选择枚举类型。__  

---
# 遇到多个构造器参数时要考虑用构建器
## 重叠构造器
```java
public class NutritionFacts {
    private final int  servingSize;
    private final int     servings;
    private final int     calories;
    private final int          fat;
    private final int       sodium;
    private final int carbohydrate;

    public NutritionFacts(int servingSize, int servings){
        this(servingSize, servings, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories){
        this(servingSize, servings, calories, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat){
        this(servingSize, servings, calories, fat, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium){
        this(servingSize, servings, calories, fat, sodium, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium, int carbohydrate){
        this.servingSize  = servingSize;
        this.servings     = servings;
        this.calories     = calories;
        this.fat          = fat;
        this.sodium       = sodium;
        this.carbohydrate = carbohydrate;
    }
}
```
### 缺点
当有许多参数的适合，客户端代码会很难编写，并且仍然较难以阅读。  

## javaBeans
```java
public class NutritionFacts {
    private int servingSize = -1;
    private int servings = -1;
    private int calories = 0;
    private int fat = 0;
    private int sodium = 0;
    private int carbohydrate = 0;

    public NutritionFacts() {}

    public void setServingSize(int val) { servingSize = val;}
    public void setServings(int val) { servingSize = val;}
    public void setCalories(int val) { servingSize = val;}
    public void setFat(int val) { servingSize = val;}
    public void setSodium(int val) { servingSize = val;}
    public void setCarbohydrate(int val) { servingSize = val;}
}
```
### 缺点
构造过程中可能__处于不一致的状态__，也就是说线程不安全。

## builder Pattern (构建器)
```java
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;

    public static class Builder {
        private final int servingSize;
        private final int servings;

        private int calories = 0;
        private int fat = 0;
        private int carbohydrate = 0;
        private int sodium = 0;

        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings = servings;
        }

        public Builder calories(int val){ calories = val; return this;}
        public Builder fat(int val){ fat = val; return this;}
        public Builder sodium(int val){ sodium = val; return this;}
        public Builder carbohydrate(int val){ carbohydrate = val; return this;}

        public NutritionFacts build() {
            return new NutritionFacts3(this);
        }
    }

    private NutritionFacts(Builder builder) {
        servingSize = builder.servingSize;
        servings = builder.servings;
        calories = builder.calories;
        fat = builder.fat;
        sodium = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
}
```
优雅的构建！！！！  
```java
NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
			.calories(100).sodium(35).carbohydrate(27).build();
```
## 总结
静态工厂和构造器有一个局限性，都不能很好的扩展到大量可选参数。一共有三个方法可以用于构造有__大量__可选构造函数的类的实例。  
如果类的构造器或者静态工厂中具有多个参数，Builder模式就是一种不错的选择。__但是在某些十分注重性能的情况下，可能就会成为问题__  

---
 
# 通过私有构造器强化不可实例化的能力
有时候想只写一个包含静态方法的类，又不想实例化，__通过将类做成抽象类来强制该类不可被实例化是行不通的。__  
这个时候,让它包含私有构造器即可!  
```java
public class UtilityClass {
    private UtilityClass(){
        throw new AssertionError();
    }
}
```
# 未完待续...
