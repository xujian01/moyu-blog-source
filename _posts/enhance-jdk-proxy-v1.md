---
title: 实现一个增强版的JDK动态代理V1.0
date: 2021-06-15 10:33:29
tags: Java
---
看了Spring AOP JDK动态代理部分的源码，想尝试借鉴Spring的思想，实现一个“增强版”的JDK动态代理。
<!-- more -->
注意：该实现方式不优雅，优化版本请参考：[实现一个增强版的JDK动态代理V2.0](https://blog.csdn.net/qq_18515155/article/details/118031761)
## 现状

```java
UserService userService = new UserServiceImpl();
MyInvocationHandler myInvocationHandler = new MyInvocationHandler(userService);
Proxy.newProxyInstance(userService.getClass().getClassLoader(),
                userService.getClass().getInterfaces(),invocationHandler);
```

```java
public class MyInvocationHandler implements InvocationHandler {
  private Object target;
  
  public MyInvocationHandler(MyInvocationHandler myInvocationHandler) {
    this.myInvocationHandler = myInvocationHandler;
  }
  @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      //增强代码
      ...
      return method.invoke(target,args);
    }
}
```

以上是大家很熟悉的创建JDK代理的代码。

我们的增强的代码全都在`InvocationHandler`的`invoke`方法中。

想象一下，我们想要对某个类进行代理，使代理类在各个不同的方面进行增强，那么`invoke`方法将会变得很重。

## 目标

分析了原生JDK代理的现状之后我们目标自然也就出来了：**根据单一职责原则，希望将不同类型的增强逻辑定义在不同的类中，然后让不同增强类的增强逻辑一起作用在目标类上，达到一种层层增强的效果。**

## 方案

### 定义拦截器

为了达成目标里提到的“单一职责原则”，需要先定义一个抽象类，这个类定义了增强逻辑，我们称它为“方法拦截器”：

```java
/**
 * @author xujian
 * 2021-05-26 10:11
 **/
public abstract class MethodInterceptor {
    /**
     * 执行增强
     * @param chain
     * @param joinPointInfo
     * @return
     * @throws Exception
     */
    public final void proceed(InterceptorChain chain, JoinPointInfo joinPointInfo) throws Exception {
        proceed(joinPointInfo);
        chain.proceed(joinPointInfo);
    }

    /**
     * 用户自定义增强逻辑
     * @param joinPointInfo
     */
    abstract void proceed(JoinPointInfo joinPointInfo);

    /**
     * 获取连接点类型
     * @return
     */
    abstract JoinPointEnum getJoinPoint();

    /**
     * 获取顺序编号，可以重写该方法修改默认排序
     * @return
     */
    public int getOrder() {return 1;}
}
```

- 可以看到有一个抽象方法`proceed`，这是给使用者自己去实现的方法，这个方法应当是要增强的逻辑。
- 还有一个`final`方法`proceed`，这个方法在“拦截器链”中被调用，**是责任链**设计模式的实现。
- 还有一个方法`getJoinPoint`，该方法是用来获取当前的“方法拦截器”的增强逻辑作用的时机，如：目标方法执行前增强，目标方法返回后执行。
- 考虑到对不同的“方法拦截器”的增强逻辑有顺序要求，还加入了一个方法`getOrder`，该方法返回一个序号，序号小越先执行增强逻辑。

### 定义拦截器链

于此同时需要定义一个连接所有“方法拦截器”的链条，称它为“拦截器链”：

```java
/**
 * 拦截器链
 * @author xujian
 * 2021-05-26 10:09
 **/
public class InterceptorChain {
    /**
     * 方法执行前拦截器列表
     */
    private List<MethodInterceptor> beforeInterceptors;
    /**
     * 方法正常返回后拦截器列表
     */
    private List<MethodInterceptor> afterReturnInterceptors;
  
    private int currentBeforeInterceptorIndex = -1;
    private int currentAfterReturnInterceptorIndex = -1;
  	/**
     * 是否已经执行过目标方法
     */
    private boolean isTargetMethodExecuted;

    public InterceptorChain(List<MethodInterceptor> interceptors) {
        //根据增强时机（连接点）分组
        Map<String, List<MethodInterceptor>> interceptorMap =
                interceptors.stream().collect(groupingBy(m -> m.getJoinPoint().getName()));
        //方法执行前增强的拦截器
        beforeInterceptors = interceptorMap.get(JoinPointEnum.BEFORE.getName());
        //按照order排序
        beforeInterceptors.sort(Comparator.comparingInt(MethodInterceptor::getOrder));
        //方法返回后增强的拦截器
        afterReturnInterceptors = interceptorMap.get(JoinPointEnum.AFTER_RETURN.getName());
        //按照order排序
        afterReturnInterceptors.sort(Comparator.comparingInt(MethodInterceptor::getOrder));
    }

  	/**
     * 执行方法
     * @param joinPointInfo
     * @return
     * @throws Exception
     */
    public Object proceed(JoinPointInfo joinPointInfo) throws Exception {
      	//方法执行前增强
        before(joinPointInfo);
        if (!isTargetMethodExecuted) {
          	//执行目标方法
            Object o = joinPointInfo.getMethod().invoke(joinPointInfo.getTarget(),joinPointInfo.getArgs());
            isTargetMethodExecuted = true;
            joinPointInfo.setReturnValue(o);
        }
      	//方法返回后增强
        afterReturn(joinPointInfo);
        return joinPointInfo.getReturnValue();
    }

    public void before(JoinPointInfo joinPointInfo) throws Exception {
        if (currentBeforeInterceptorIndex == beforeInterceptors.size() - 1) {
            return;
        }
        MethodInterceptor methodInterceptor = beforeInterceptors.get(++currentBeforeInterceptorIndex);
        methodInterceptor.proceed(this,joinPointInfo);
    }

    public void afterReturn(JoinPointInfo joinPointInfo) throws Exception {
        if (currentAfterReturnInterceptorIndex == afterReturnInterceptors.size() - 1) {
            return;
        }
        MethodInterceptor methodInterceptor = afterReturnInterceptors.get(++currentAfterReturnInterceptorIndex);
        methodInterceptor.proceed(this,joinPointInfo);
    }
}
```

- “拦截器链"的`proceed`里面完成对目标方法的“前置”、“后置”增强。
- `brfore`和`afterReturn`分别是“前置”和“后置”增强的具体实现，里面会调用“方法拦截器”的`proceed`方法，**这也是实现责任链**设计模式的一部分。

### 定义InvocationHandler

因为我们是对原生JDK代理的增强，所以还是需要依赖原生的JDK代理功能：

```java
/**
 * @author xujian
 * 2021-05-26 10:58
 **/
public class EnhanceInvocationHandler implements InvocationHandler {
    //目标对象
    private Object target;
  	//拦截器链
    private InterceptorChain chain;

    public EnhanceInvocationHandler(Object target, List<MethodInterceptor> interceptors) {
        this.target = target;
        this.chain = new InterceptorChain(interceptors);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        JoinPointInfo joinPointInfo = JoinPointInfo.Builder.newBuilder()
                .method(method).target(target).proxy(proxy).args(args)
                .build();
      	//执行方法增强
        return chain.proceed(joinPointInfo);
    }
}
```

- 它持有一个“拦截器链”的引用，以便调用其执行增强的方法。
- 它是一个默认且固定的`InvocationHandler`实现，不需要使用者再去实现`InvocationHandler`。

### 定义增强的代理生成类

为了让使用者用类似使用原生的JDK代理的方式使用增强的JDK代理，需要定义一个代理生成类：

```java
/**
 * 增强的jdk代理
 * @author xujian
 * 2021-05-26 10:06
 **/
public class EnhanceProxy {
    /**
     * 获取代理类
     * @param loader
     * @param interfaces
     * @param h
     * @return
     */
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                  EnhanceInvocationHandler h) {
        return Proxy.newProxyInstance(loader,interfaces,h);
    }
}
```

- 该类有一个静态的`newProxyInstance`方法，这个和原生的JDK代理类`Proxy.newProxyInstance(...)`很类似，但是它最后一个参数只接受`EnhanceInvocationHandler`类型，这个不同于原生的JDK代理。因为`EnhanceInvocationHandler`是作为内部唯一默认且固定的实现而存在。



## 测试

定义好了核心组件之后，测试一下。

> 该测试仅验证是否能够达到预期的效果。

**预期效果**

有一个打印方法`print`，想要在该方法执行之前做两层增强，以后在该方法返回后执行两层增强。

首先，需要继承`MethodInterceptor`来编写增强逻辑。

`BeforeMethodInterceptor`

```java
/**
 * @author xujian
 * 2021-05-26 11:35
 **/
public class BeforeMethodInterceptor extends MethodInterceptor {


    /**
     * 用户自定义增强逻辑
     *
     * @param joinPointInfo
     */
    @Override
    void proceed(JoinPointInfo joinPointInfo) {
        System.out.println("----before");
    }

    /**
     * 获取连接点类型
     *
     * @return
     */
    @Override
    JoinPointEnum getJoinPoint() {
        return JoinPointEnum.BEFORE;
    }

    /**
     * 获取顺序编号，可以重写该方法修改默认排序
     *
     * @return
     */
    @Override
    public int getOrder() {
        return 2;
    }
}
```

`BeforeBeforeMethodInterceptor`

```java
/**
 * @author xujian
 * 2021-05-26 11:35
 **/
public class BeforeBeforeMethodInterceptor extends MethodInterceptor {

    /**
     * 用户自定义增强逻辑
     *
     * @param joinPointInfo
     */
    @Override
    void proceed(JoinPointInfo joinPointInfo) {
        System.out.println("----beforebefore");
    }

    /**
     * 获取连接点类型
     *
     * @return
     */
    @Override
    JoinPointEnum getJoinPoint() {
        return JoinPointEnum.BEFORE;
    }
}
```

`AfterReturnMethodInterceptor`

```java
/**
 * @author xujian
 * 2021-05-26 11:35
 **/
public class AfterReturnMethodInterceptor extends MethodInterceptor {


    /**
     * 用户自定义增强逻辑
     *
     * @param joinPointInfo
     */
    @Override
    void proceed(JoinPointInfo joinPointInfo) {
        System.out.println("----after");
    }

    /**
     * 获取连接点类型
     *
     * @return
     */
    @Override
    JoinPointEnum getJoinPoint() {
        return JoinPointEnum.AFTER_RETURN;
    }
}
```

`AfterAfterReturnMethodInterceptor`

```java
/**
 * @author xujian
 * 2021-05-26 11:35
 **/
public class AfterAfterReturnMethodInterceptor extends MethodInterceptor {

    /**
     * 用户自定义增强逻辑
     *
     * @param joinPointInfo
     */
    @Override
    void proceed(JoinPointInfo joinPointInfo) {
        System.out.println("----afterafter");
        joinPointInfo.setReturnValue("我是"+this.getClass().getSimpleName()+"拦截器修改以后的返回值");
    }

    /**
     * 获取连接点类型
     *
     * @return
     */
    @Override
    JoinPointEnum getJoinPoint() {
        return JoinPointEnum.AFTER_RETURN;
    }
}
```

目标类：

```java
/**
 * @author xujian
 * 2021-05-26 11:32
 **/
public class UserServiceImpl implements UserService {
    @Override
    public String print() {
        System.out.println("----UserService的打印方法");
        return "我是返回值";
    }
}
```

测试类：

```java
@Test
    public void test01() {
        UserService userService = new UserServiceImpl();

        //为目标类添加方法拦截器即要增强的逻辑，请注意添加顺序
        List<MethodInterceptor> interceptors = new ArrayList<>();
        interceptors.add(new BeforeMethodInterceptor());
        interceptors.add(new BeforeBeforeMethodInterceptor());
        interceptors.add(new AfterReturnMethodInterceptor());
        interceptors.add(new AfterAfterReturnMethodInterceptor());

        EnhanceInvocationHandler enhanceInvocationHandler = new EnhanceInvocationHandler(userService,interceptors);

        //生成代理对象
        UserService userService1 = (UserService) EnhanceProxy.newProxyInstance(userService.getClass().getClassLoader(),
                userService.getClass().getInterfaces(),enhanceInvocationHandler);

        System.out.println(userService1.print());
    }
```

> 注意上面“方法拦截器”的添加顺序。

执行结果

```
----beforebefore
----before
----UserService的打印方法
----after
----afterafter
方法返回结果：我是AfterAfterReturnMethodInterceptor拦截器修改以后的返回值
```

- 结果显示先进行了方法执行前增强，再执行目标方法，再执行方法返回后增强。
- 注意到方法前置增强的顺序和“方法拦截器”的添加顺序不一致：before本应该在beforebefore之前打印才对。这是因为`BeforeMethodInterceptor`重写了`getOrder`方法，返回了2，从而调整了顺序。
- 除了增强外，还对方法的返回值进行了修改：返回的数据从“我是返回值”变为了“我是AfterAfterReturnMethodInterceptor拦截器修改以后的返回值”。

## 总结

- 本文介绍了如何对原生JDK动态代理进行简单的增强；
- 根据**单一职责原则**将不同的增强逻辑放到不同的“方法拦截器”里；
- 使用**责任链模式**实现了对目标类的“层层”增强；
- 除了基于Spring AOP JDK代理的“层层”增强思路，还可以借鉴Mybatis插件实现机制的思路：为对象生成的“层层”代理对象来达到“层层”增强的效果。可以参考：[这么强大的Mybatis插件机制原来就是这？](https://blog.csdn.net/qq_18515155/article/details/116991153)来了解Mybatis的插件机制。
---
相关代码请参考：[https://gitee.com/xujian01/blogcode/tree/master/src/main/java/enhancejdkproxy](https://gitee.com/xujian01/blogcode/tree/master/src/main/java/enhancejdkproxy) 