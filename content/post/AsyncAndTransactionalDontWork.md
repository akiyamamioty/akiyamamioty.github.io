---
title: "Spring @Async/@Transactional失效原因及解决方案"
date: 2019-09-14T16:56:56+08:00
draft: false
tags: ["Spring", "AOP", "动态代理", "Java"]
categories: ["Spring", "Java"]
---

之前提到实现AOP的方法有动态代理、编译期，类加载期织入等等，Spring实现AOP的方法则就是利用了动态代理机制，正因如此，才会导致某些情况下@Async和@Transactional不生效。


## @Async正常使用方法
我们已@Aysnc为例，在Spring boot中使用Async很简单：
```
@EnableAsync //添加此注解开启异步调用
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

当某些任务执行时间较长，且客户端不需要及时获取结果（如调用第三方API），只要在需要异步调用的任务上添加 ```@Async```即可，如：

```
@Async
void asyncTask(String keyword) {
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        //logger
        //error tracking
    }
    System.out.println(keyword);
}
```

这样```asyncTask```就会异步执行。 
## @Async使用失效情况
然而，如果在同一个Class内，出现下面这样的情况，先调用一个非异步任务：

```
private void noAsyncTask(String keyword){
    asyncTask(keyword); //该方法内再调用异步方法
}

@Async
void asyncTask(String keyword) {
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        //logger
        //error tracking
    }
    System.out.println(keyword);
}
```
**此时，@Async是没有生效的**。 

## 失效原因
**原因就是@Async和@Transaction利用了动态代理机制。**

当Spring发现@Transactional或者@Async时，会自动生成一个ProxyObject，如：

 <div align=center>![ProxyObject](/image/async/1.png)</div>

此时调用Class.transactionTask会调用ProxyClass.产生事务操作。
然而当Class里的一个非事务方法调用了事务方法，ProxyClass是这样的：
 <div align=center>![ProxyObject](/image/async/2.png)</div>

到这里应该可以看明白了，如果调用了noTransactionTask方法，最终会调用到Class.transactionTask，而这个方法是不带有任何Transactional的信息的，也就是@Transactional根本没有生效哦。
简单来说就是： **同一个类内这样调用的话，只有第一次调用了动态代理生成的ProxyClass，之后一直用的是不带任何切面信息的方法本身。**

## 解决方案
知道了原因，处理方法也特别简单，就是让noTransactionTask里依旧调用ProxyClass的transactionTask方法：只需要显示利用Spring暴露的AopContext即可。代码如下：

```
private void noAsyncTask(String keyword){
    // 注意这里 调用了代理类的方法
    ((YourClass) AopContext.currentProxy()).asyncTask(keyword);
}

@Async
void asyncTask(String keyword) {
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        //logger
        //error tracking
    }
    System.out.println(keyword);
}
```
记得要在Class上加上```@EnableAspectJAutoProxy(exposeProxy = true)```来暴露AOP的Proxy对象才行，否则会报错。

或者就可以把这样的方法放到另外一个类里，不要产生类里一个非异步/非事务方法，调用了异步/事务方法，不过大家协同开发同一个文件的话，谁能保证没有人这样调用呢？总而言之无论什么方案，都是使得调用ProxyObject的方法。





