---
title: "Java Synchronized 关键字的用法"
date: 2019-12-04T21:50:00+08:00
tags: ["JAVA", "JVM"]
categories: ["JAVA", "JVM"]
---

本文简单介绍 Java **Synchronized**关键字的用法和一小部分原理，不会涉及过多底层细节。

### 简述

Java 的 Synchronized 同步机制是 Java 的第一个用于同步访问多个线程共享的对象的机制。但是最初 Synchronized 同步机制不是很高效，这就是为什么 Java 5 之后提供了一整套并发类来帮助我们实现比 Synchronized 更加细粒度的并发控制的原因。但是在之后Java版本中，对于synchronized也做了非常多的优化。

### synchronized 关键字

Java 中的 Synchronized 同步块都用 synchronized 关键字标记。Java 中的同步块是指在某些对象上同步。在同一个对象上同步的所有代码块只能在其内部同时执行一个线程。尝试进入同步块的所有其他线程将被阻止，直到同步块内的线程退出该块为止。

Synchronized 关键字可用于标记四种不同类型的块：

- 实例方法
- 静态方法
- 实例方法中的代码块
- 静态方法中的代码块

这些块在不同的对象上同步。需要哪种类型的同步块取决于具体情况。

### synchronized实例方法

```
public class MyClass {

  private int number = 0;

  public synchronized void add(int value){
      this.number += value;
  }
}
```
在add（）方法声明中使用了synchronized关键字，表明这个方法是同步的。

Java中的同步实例方法在***拥有该方法的实例（对象）上同步***。每个MyClass实例只能有一个线程可以在同步实例方法（add方法）中执行。不过，如果你new了多个MyClass实例，每个实例的add方法都在它们各自的实例上同步。

如果存在多个实例，**则每个实例可以一次在一个同步实例方法中执行一个线程。**

如果一个实例存在多个同步实例方法，则**每个实例只有一个线程可以执行一个方法**。因此，下面的代码中，只有一个线程可以在两个同步(add/substract)方法中的一个内部执行。即**每个实例总共一个线程**：
```
public class MyClass {

  private int number = 0;

  public synchronized void add(int value){
      this.number += value;
  }
  public synchronized void subtract(int value){
      this.number -= value;
  }
}
```


### synchronized 静态方法
静态方法被标记为同步方法，就像和用synchronized关键字的实例方法一样：
```
public static MyStaticClass{

  private static int number = 0;

  public static synchronized void add(int value){
      number += value;
  }
}
```

同样的，synchronized关键字表明了add（）方法是同步的。

同步静态方法在***同步静态方法所属类的类对象上同步***。由于每个类的JVM中仅存在一个类对象，因此**在同一类中的静态同步方法内只能执行一个线程。**

如果一个类包含多个静态同步方法，则只能在一个方法中同时执行一个线程。就如同synchronized实例方法中的add/substract方法一样，不再过多举例。

### 实例方法中的synchronized代码块
有时候我们不必同步整个方法，那么最好只同步一部分方法。synchronized也提供的同步方法中代码块的功能。

这是未被synchronized声明的Java方法中的synchronized代码块:

```
public void add(int value){
  synchronized(this){
     this.number += value;   
  }
}
```
使用 synchronized(this)将代码块标记为同步代码块。这段代码将像执行同步方法一样执行。

要注意Java同步代码块是如何在括号中使用对象。在示例代码中中，**使用了“ this”，这是调用add方法的实例**。同步代码块在括号中使用的对象称为监视（monitor）对象。意思是：**在Monitor对象上，这段代码是同步的**。同步实例方法将其所属的Class用作monitor对象。

在同一监视对象上同步的代码块内只能执行一个线程。
此外，这段代码和synchronized实例方法的add（）方法是等效的。

### 静态方法中的synchronized代码块

synchronized也可以在静态方法内部使用。下面例子中的方法在该方法所属的MyClass类的类对象上同步：

```
public static void add(int value){
     synchronized(MyClass.class){
        this.number += value; 
     }
}
```
如果同时存在synchronized的静态方法，和静态方法中的synchronized代码块，同时只能有一个线程可以在任意一个方法的内部执行。

### 同步（Synchronized）与数据可见性（Data Visibility）
如果我们不使用Synchronized关键字（或Java volatile关键字），则无法保证当一个线程更改与其他线程共享的变量的值时，其他线程可以看到更改后的值。开发人员无法控制或保证何时将一个线程保留在CPU寄存器中的变量提交(commit)到主存储器，也无法保证何时其他线程从主存储器刷新(refresh) CPU寄存器中的变量。

同步关键字可以改变这一点。当线程进入同步块时，它将刷新该线程可见的所有变量的值。当线程退出同步块时，对该线程可见的变量的所有更改都将提交给主内存。这有一点类似于volatile关键字的工作方式。

### 需要在哪些对象上同步？
如本在本文多次提到的，同步块必须在某个对象上同步。实际上，我们可以选择任何要同步的对象，但是建议不要在String对象或任何原始类型包装类对象上进行同步，因为编译器可能会优化这些对象，以便在我们在不同的地方使用相同的实例。

比如"OK", Integer.valueOf(1), Java编译器或者Java库会实际上会使用相同的对象。如果你以为用了不同的对象，那么程序可能产生你不想要的行为

所以为了安全起见，在this或new Object()上进行同步。 Java编译器/JVM/Java库不会在内部缓存或重用它们。

### 其他
后续会继续更新一些synchronized的限制和替代方案，还会继续深挖一下底层细节。
