---
title: 实现一个增强版的JDK动态代理V2.0
date: 2021-06-19 10:33:29
tags: Java
---
之前有个文章实现了一个简单的增强版JDK动态代理：[实现一个增强版的JDK动态代理V1.0](https://blog.csdn.net/qq_18515155/article/details/117334651)。
但是当时只是实现了方法前增强和方法返回后增强的效果，至少还有如下两个不足：
1）前、后增强是通过写死执行增强的位置来实现；
2）没有实现环绕增强；
<!-- more -->
## 分析之前实现的不足

### 1）前、后增强是通过写死执行增强的位置来实现

先来回顾一下之前实现的代码

```java
public Object proceed(JoinPointInfo joinPointInfo) throws Exception {
      	//1）方法执行前增强
        before(joinPointInfo);
  			// 判断目标方法是否已经执行过了
        if (!isTargetMethodExecuted) {
          	//执行目标方法
            Object o = joinPointInfo.getMethod().invoke(joinPointInfo.getTarget(),joinPointInfo.getArgs());
            isTargetMethodExecuted = true;
            joinPointInfo.setReturnValue(o);
        }
      	//2）方法返回后增强
        afterReturn(joinPointInfo);
        return joinPointInfo.getReturnValue();
    }
```

可以看到上面1）、2）两处，是通过固定其执行的位置，让before拦截器在真正的方法执行前执行，让afterReturn拦截器在真正的方法执行后执行。

这样做相当于是写死了一个模版，不优雅并且后期不容易扩展（甚至都不能满足环绕增强的需求）

### 2）没有实现环绕增强

最常用的环绕增强功能没有实现，且在当前的设计下很难实现，必须重新思考多层拦截增强的本质进行重新设计。

## 优化改进

>  在参考了Spring AOP解决方案以后，会发现：
>
> - 不同类型拦截器的增强就是方法调用，**栈帧**入栈和出栈的一个过程；
>
> - 而相同类型多个拦截器的增强就是**责任链模式**的调用。

1、首先将`MethodInterceptor`抽象为一个接口：

```java
/**
     * 执行增强
     * @param chain
     * @param joinPointInfo
     * @return
     * @throws Exception
     */
    Object proceed(InterceptorChain chain, JoinPointInfo joinPointInfo) throws Exception;
    /**
     * 用户自定义增强逻辑
     * @param joinPointInfo
     */
    void proceed(JoinPointInfo joinPointInfo) throws Exception;

    /**
     * 获取连接点类型
     * @return
     */
    JoinPointEnum getJoinPoint();

    /**
     * 获取顺序编号，可以重写该方法修改默认排序
     * @return
     */
    default int getOrder() {return 1;}
```

2、分别针对方法执行前增强、方法返回后增强实现基于`MethodInterceptor`的抽象类：

```java
/**
 * @author xujian
 * 2021-05-26 10:11
 **/
public abstract class AbstractBeforeMethodInterceptor implements MethodInterceptor {
    /**
     * 执行增强
     * @param chain
     * @param joinPointInfo
     * @return
     * @throws Exception
     */
    @Override
    public final Object proceed(InterceptorChain chain, JoinPointInfo joinPointInfo) throws Exception {
        proceed(joinPointInfo);
        return chain.proceed(joinPointInfo);
    }
	
	/**
     * 获取连接点类型
     *
     * @return
     */
    @Override
    public final JoinPointEnum getJoinPoint() {
        return JoinPointEnum.BEFORE;
    }
}

/**
 * @author xujian
 * 2021-05-26 10:11
 **/
public abstract class AbstractAfterReturnMethodInterceptor implements MethodInterceptor {
    /**
     * 执行增强
     * @param chain
     * @param joinPointInfo
     * @return
     * @throws Exception
     */
    @Override
    public final Object proceed(InterceptorChain chain, JoinPointInfo joinPointInfo) throws Exception {
        Object o = chain.proceed(joinPointInfo);
        proceed(joinPointInfo);
        return o;
    }
	
	/**
     * 获取连接点类型
     *
     * @return
     */
    @Override
    public final JoinPointEnum getJoinPoint() {
        return JoinPointEnum.AFTER_RETURN;
    }
}
```

> 方法环绕增强不需要实现对应的抽象类，增强逻辑完全由使用者实现，参考Spring AOP Aspect。

3、修改InterceptorChain执行增强的逻辑：

```java
/**
 * 拦截器链
 * @author xujian
 * 2021-05-26 10:09
 **/
public class InterceptorChain {
    /**
     * 拦截器列表
     */
    private List<MethodInterceptor> methodInterceptors = new ArrayList<>();
    private int currentInterceptorIndex = -1;

    public InterceptorChain(List<MethodInterceptor> interceptors) {
        //根据增强时机（连接点）分组
        Map<String, List<MethodInterceptor>> interceptorMap =
                interceptors.stream().collect(groupingBy(m -> m.getJoinPoint().getName()));
        //方法执行前增强的拦截器
        List<MethodInterceptor> beforeInterceptors = interceptorMap.get(JoinPointEnum.BEFORE.getName());
        //按照order排序
        beforeInterceptors.sort(Comparator.comparingInt(MethodInterceptor::getOrder));
        //方法返回后增强的拦截器
        List<MethodInterceptor> afterReturnInterceptors = interceptorMap.get(JoinPointEnum.AFTER_RETURN.getName());
        //按照order排序
        afterReturnInterceptors.sort(Comparator.comparingInt(MethodInterceptor::getOrder));
        //方法环绕增强的拦截器
        List<MethodInterceptor> aroundInterceptors = interceptorMap.get(JoinPointEnum.AROUND.getName());
        //按照order排序
        aroundInterceptors.sort(Comparator.comparingInt(MethodInterceptor::getOrder));
        //按照around->before->afterReturn的顺序添加到methodInterceptors
        methodInterceptors.addAll(aroundInterceptors);
        methodInterceptors.addAll(beforeInterceptors);
        methodInterceptors.addAll(afterReturnInterceptors);
    }

    /**
     * 执行方法
     * @param joinPointInfo
     * @return
     * @throws Exception
     */
    public Object proceed(JoinPointInfo joinPointInfo) throws Exception {
        if (currentInterceptorIndex == methodInterceptors.size() - 1) {
            //执行目标方法
            return joinPointInfo.getMethod().invoke(joinPointInfo.getTarget(),joinPointInfo.getArgs());
        }
        MethodInterceptor methodInterceptor = methodInterceptors.get(++currentInterceptorIndex);
        return methodInterceptor.proceed(this,joinPointInfo);
    }
}
```

可以和旧版做个对比：

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

可以发现新版相比旧版`proceed()`方法明显优雅简洁了很多。

4、自定义拦截增强器

对之前的`AfterAfterReturnMethodInterceptor、AfterReturnMethodInterceptor、BeforeBeforeMethodInterceptor、BeforeMethodInterceptor`做修改如下：

```java
/**
 * @author xujian
 * 2021-05-26 11:35
 **/
public class AfterAfterReturnMethodInterceptor extends AbstractAfterReturnMethodInterceptor {

    /**
     * 用户自定义增强逻辑
     *
     * @param joinPointInfo
     */
    @Override
    public void proceed(JoinPointInfo joinPointInfo) {
        System.out.println("----afterafter");
    }
}

/**
 * @author xujian
 * 2021-05-26 11:35
 **/
public class AfterReturnMethodInterceptor extends AbstractAfterReturnMethodInterceptor {


    /**
     * 用户自定义增强逻辑
     *
     * @param joinPointInfo
     */
    @Override
    public void proceed(JoinPointInfo joinPointInfo) {
        System.out.println("----after");
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

/**
 * @author xujian
 * 2021-05-26 11:35
 **/
public class BeforeBeforeMethodInterceptor extends AbstractBeforeMethodInterceptor {


    /**
     * 用户自定义增强逻辑
     *
     * @param joinPointInfo
     */
    @Override
    public void proceed(JoinPointInfo joinPointInfo) {
        System.out.println("----beforebefore");
    }
}

/**
 * @author xujian
 * 2021-05-26 11:35
 **/
public class BeforeMethodInterceptor extends AbstractBeforeMethodInterceptor {

    /**
     * 用户自定义增强逻辑
     *
     * @param joinPointInfo
     */
    @Override
    public void proceed(JoinPointInfo joinPointInfo) {
        System.out.println("----before");
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

除此之外，新增一个环绕拦截器，该拦截器直接实现`MethodInterceptor`接口：

```java
/**
 * @author xujian
 * 2021-05-26 11:35
 **/
public class AroundMethodInterceptor implements MethodInterceptor {
    /**
     * 执行增强
     *
     * @param chain
     * @param joinPointInfo
     * @return
     * @throws Exception
     */
    @Override
    public Object proceed(InterceptorChain chain, JoinPointInfo joinPointInfo) throws Exception {
        System.out.println("----around-before");
        Object o = chain.proceed(joinPointInfo);
        System.out.println("----around-after");
        return o;
    }

    /**
     * 用户自定义增强逻辑
     *
     * @param joinPointInfo
     */
    @Override
    public void proceed(JoinPointInfo joinPointInfo) throws Exception {

    }

    /**
     * 获取连接点类型
     *
     * @return
     */
    @Override
    public JoinPointEnum getJoinPoint() {
        return JoinPointEnum.AROUND;
    }
}
```

5、修改测试方法

拦截器链除了之前的四个拦截器，本次再加上新增的`AroundMethodInterceptor`拦截器：

```java
interceptors.add(new AroundMethodInterceptor());
```

6、测试验证

对于before、afterReturn、around这三种拦截增强的执行结果预期要和Spring AOP Aspect一样：

1、先执行`around#System.out.println("----around-before")；`

2、再执行`before#System.out.println("----before")；`

3、再执行业务方法`userService#print()；`

4、再执行`afterReturn#System.out.println("----after")；`

5、最后再执行`around#System.out.println("----around-after")；`

由于多个相同类型的拦截器会根据order字段排序，所以**真正期望的结果是**：

1、先执行`AroundMethodInterceptor#System.out.println("----around-before")；`

2、再执行`BeforeBeforeMethodInterceptor#System.out.println("----beforebefore")；`

3、再执行`BeforeMethodInterceptor#System.out.println("----before")；`

4、再执行业务方法`userService#print()；`

5、再执行`AfterAfterReturnMethodInterceptor#System.out.println("----afterafter")；`

6、再执行`AfterReturnMethodInterceptor#System.out.println("----after")；`

7、最后再执行`AroundMethodInterceptor#System.out.println("----around-after")；`

实际测试结果如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210618182644191.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
和预期结果一致。

整个执行流程可以在参考一下下面的时序图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210618182704902.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)

---

相关代码请参考：[https://gitee.com/xujian01/blogcode/tree/master/src/main/java/enhancejdkproxy](https://gitee.com/xujian01/blogcode/tree/master/src/main/java/enhancejdkproxy)


