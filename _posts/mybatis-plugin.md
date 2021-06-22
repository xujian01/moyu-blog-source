---
title: 这么强大的Mybatis插件机制原来就是这？
date: 2021-05-13 10:33:29
tags: Mybatis
---
Mybatis开发中经常会用到pagehelper分页插件，除此之外还有慢sql上报等各种各样的插件，那么Mybatis是如何来实现如此强大的插件机制呢？一起来看看吧。
<!-- more -->
## Mybatis插件机制介绍

MyBatis 允许你在映射语句执行过程中的某一点进行拦截调用。默认情况下，MyBatis 允许使用插件来拦截的方法调用包括：

- Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)

执行器，提供操作数据库的接口。

- ParameterHandler (getParameterObject, setParameters)

参数处理器，设置sql的参数。

- ResultSetHandler (handleResultSets, handleOutputParameters)

结果集处理器，处理从数据库查询的结果集，封装成对象等。

- StatementHandler (prepare, parameterize, batch, update, query)

语法处理器，真正去执行数据库CRUD。

他们的引用关系如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210518153023684.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)
可见其插件是基于**方法拦截**来实现的！我们姑且猜测是和AOP有关，到底对不对呢，往下看就知道了。

## 自定义一个Mybatis插件

自定义一个插件是如此简单，以拦截`ResultSetHandler`为例，仅需要两步（Mybatis版本：3.5.6）：

步骤一：实现Interceptor接口，如下所示

```java
package mybatisplugin;

import lombok.extern.slf4j.Slf4j;
import org.apache.ibatis.executor.resultset.ResultSetHandler;
import org.apache.ibatis.plugin.Interceptor;
import org.apache.ibatis.plugin.Intercepts;
import org.apache.ibatis.plugin.Invocation;
import org.apache.ibatis.plugin.Signature;
import java.sql.Statement;

/**
 * @author xujian
 * 2021-05-11 15:22
 **/
@Intercepts(@Signature(type = ResultSetHandler.class,//指定你要拦截是哪个对象
        method = "handleResultSets",//指定你要拦截的是哪个方法
        args = {Statement.class}))//考虑到方法重载，还需要指定参数列表
@Slf4j
public class MyPlugin implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        log.info("----插件拦截到handleResultSets----");
        //实行原始的方法调用
        return invocation.proceed();
    }
}
```

步骤二：在Mybatis的配置文件加入自定义插件的配置，如下所示

```xml
	<plugins>
        <plugin interceptor="mybatisplugin.MyPlugin"></plugin>
    </plugins>
```

这样，当执行了数据库查询操作，调用`ResultSetHandler#handleResultSets`封装返回结果集之前会打印日志，如下图所示
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210518153234753.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)
## 插件执行原理分析

### 插件定义

实现`Interceptor`，其源码如下

```java
/**
 * 拦截器接口
 *
 * @author Clinton Begin
 */
public interface Interceptor {

    /**
     * 拦截方法，自定义插件需要实现的方法
     *
     * @param invocation 调用信息
     * @return 调用结果
     * @throws Throwable 若发生异常
     */
    Object intercept(Invocation invocation) throws Throwable;

  	/**
     * 应用插件。如应用成功，则会创建目标对象的代理对象
     *
     * @param target 目标对象
     * @return 应用的结果对象，可以是代理对象，也可以是 target 对象，也可以是任意对象。具体的，看代码实现
     */
    default Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

 		/**
     * 设置插件属性
     *
     * @param <plugin>标签的properties 属性
     */
    default void setProperties(Properties properties) {
    }
}
```

插件是定义好了，那插件是在什么时候生效的呢？

### 插件初始化

当在配置文件配置上自定义插件以后，mybatis在初始化解析配置文件的时候，就会解析<plugin>标签。

`XMLConfigBuilder#parseConfiguration`

```java
private void parseConfiguration(XNode root) {
        try {
            ...
            // 解析 <plugins /> 标签
            pluginElement(root.evalNode("plugins"));
            ...
        } catch (Exception e) {
            throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
        }
    }
```



去到真正解析<plugin>标签的方法`XMLConfigBuilder#pluginElement`

```java
private void pluginElement(XNode parent) throws Exception {
        if (parent != null) {
            // 遍历 <plugins /> 标签
            for (XNode child : parent.getChildren()) {
              	//从xml的interceptor属性中解析出插件的名称（包名+类名即全限定名）
                String interceptor = child.getStringAttribute("interceptor");
              	//1、从xml配置文件解析出来插件的配置
                Properties properties = child.getChildrenAsProperties();
                //2、创建 Interceptor 对象，并设置属性
                Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
              	//3、给插件对象设置上插件配置的属性
                interceptorInstance.setProperties(properties);
                //4、添加到 configuration 中
                configuration.addInterceptor(interceptorInstance);
            }
        }
    }
```

注意到上面的第2步，该步骤是通过反射创建出插件对象的实例，也就是MyPlugin对象。

跟踪`resolveClass`方法，最后发现实际上是调用的`TypeAliasRegistry#resolveAlias`

具体来看代码

```java
/**
*参数是插件的全限定名
返回插件的Class对象
**/
public <T> Class<T> resolveAlias(String string) {
        try {
            if (string == null) {
                return null;
            }
            // issue #748
            // 转换成小写
            String key = string.toLowerCase(Locale.ENGLISH);
            Class<T> value;
            if (TYPE_ALIASES.containsKey(key)) {
            //首先，从 TYPE_ALIASES（别名注册表） 中获取，如果配置了别名，
          	//那么配置插件的时候就不需要配置全限定名，只需要配置类名即可，
          	//那么就会走到该分支
                value = (Class<T>) TYPE_ALIASES.get(key);
            // 其次，直接获得对应类
            } else {
                value = (Class<T>) Resources.classForName(string);
            }
            return value;
        } catch (ClassNotFoundException e) { // 异常
            throw new TypeException("Could not resolve type alias '" + string + "'.  Cause: " + e, e);
        }
    }
```

经过上述过程就拿到了插件的实例对象，然后走到步骤4，该步骤是将插件设置到全局配置类`Configuration`中。

跟踪`addInterceptor`方法，发现其最终调用的是`InterceptorChain#addInterceptor`，而`Configuration`中就持有`InterceptorChain`的引用。

```java
/**
 * 拦截器链
 *
 * @author Clinton Begin
 */
public class InterceptorChain {

    /**
     * 拦截器数组
     */
    private final List<Interceptor> interceptors = new ArrayList<>();

    /**
     * 应用所有插件
     *
     * @param target 目标对象
     * @return 应用结果
     */
    public Object pluginAll(Object target) {
        for (Interceptor interceptor : interceptors) {
            target = interceptor.plugin(target);
        }
        return target;
    }

    public void addInterceptor(Interceptor interceptor) {
        interceptors.add(interceptor);
    }

    public List<Interceptor> getInterceptors() {
        return Collections.unmodifiableList(interceptors);
    }

}
```

`InterceptorChain`内部维护了一个`Interceptor`列表，用来存放所有自定义的插件。

### 插件如何生效

注意到上面自定义的插件类上有如下注解

```java
@Intercepts(@Signature(type = ResultSetHandler.class,//指定你要拦截是哪个对象
        method = "handleResultSets",//指定你要拦截的是哪个方法
        args = {Statement.class}))//考虑到方法重载，还需要指定参数列表
```

该注解声明了该插件生效的时机：mybatis在调用`ResultSetHandler#handleResultSets(Statement stmt)`方法时会执行插件逻辑。

实际上`ResultSetHandler`是一个接口，它的默认实现类是`DefaultResultSetHandler`。

我们知道，mybatis最终是通过`ResultSetHandler`来处理从数据库查询的结果的，

那就来找到创建`ResultSetHandler`的地方

```java
// 创建 ResultSetHandler 对象
    public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler,
                                                ResultHandler resultHandler, BoundSql boundSql) {
        // 1、创建 DefaultResultSetHandler 对象
        ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
        // 2、应用插件
        resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
        return resultSetHandler;
    }
```

重点关注步骤2，在步骤1创建了一个`DefaultResultSetHandler`创建了一个默认的`ResultSetHandler`实现以后，调用了`InterceptorChain#pluginAll`方法，继续跟踪该方法，最终发现是遍历`InterceptorChain`保存的所有的插件，然后逐个调用插件的`plugin`方法

```java
/**
     * 应用所有插件
     *
     * @param target 目标对象
     * @return 应用结果
     */
    public Object pluginAll(Object target) {
        for (Interceptor interceptor : interceptors) {
            target = interceptor.plugin(target);
        }
        return target;
    }
```

紧接着再来看看`Interceptor#plugin`方法

```java
Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }
```

内部调用了`Plugin`类的静态`wrap`方法

高能来了！高能来了！高能来了！

```java
/**
     * 创建目标类的代理对象
     *
     * @param target 目标类
     * @param interceptor 拦截器对象
     * @return 代理对象
     */
    public static Object wrap(Object target, Interceptor interceptor) {
        // 获得拦截的方法映射，哪个类的哪些方法需要被拦截
        Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
        // 获得目标类的类型
        Class<?> type = target.getClass();
        // 获得目标类的接口集合
        Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
        // 若有接口，则创建目标对象的 JDK Proxy 对象
        if (interfaces.length > 0) {
            return Proxy.newProxyInstance(
                    type.getClassLoader(),
                    interfaces,
                    new Plugin(target, interceptor, signatureMap)); // 因为 Plugin 实现了 InvocationHandler 接口，所以可以作为 JDK 动态代理的调用处理器
        }
        // 如果没有，则返回原始的目标对象
        return target;
    }
```

`Proxy.newProxyInstance(type.getClassLoader(),interfaces,new Plugin(target, interceptor, signatureMap));`

这行代码难道你不熟悉吗，没错，这正是**JDK动态代理**的写法。

> 从这里可以看出来mybatis创建的ResultSetHandler对象其实是经过层层代理过的对象。

那既然是JDK动态代理，那就应该有对`InvocationHandler`的实现，是谁呢？

看看`Plugin`这个类的定义`public class Plugin implements InvocationHandler`会发现就是它了。

紧接着来看看`InvocationHandler`最重要的`invoke`方法的实现

```java
@Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            // 当前方法所属的类有哪些需要被拦截的方法
            Set<Method> methods = signatureMap.get(method.getDeclaringClass());
            if (methods != null && methods.contains(method)) {
                // 1、如果当前方法包含在要被拦截的方法之内，则拦截处理该方法
                return interceptor.intercept(new Invocation(target, method, args));
            }
            // 2、如果不是，则调用原方法
            return method.invoke(target, args);
        } catch (Exception e) {
            throw ExceptionUtil.unwrapThrowable(e);
        }
    }
```

1. 如果当前方法要被拦截，那就调用插件的`intercept`方法；
2. 如果当前方法不需要被拦截，那就直接调用原始对象的对应方法；

看到这里就豁然开朗了，原来自定义插件时实现的`intercept`方法是在这里被调用了。


## 整体思路
现在来整理一下思路：

1、将自定义插件保存到**插件链**中；
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210518153059983.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)
2、使用插件链中的插件层层代理`ResultSetHandler`；
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210518153124699.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)
3、调用插件的`intercept`方法；
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210609193640498.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)
## 总结

经过上面的分析，发现Mybatis的插件机制主要还是依赖于JDK动态代理，更抽象一点说是依赖于AOP思想，其核心就是“拦截+增强”，那其实除了Mybatis的插件机制，Skywalking的插件机制也是基于这种思想实现的，可以参考：[Skywalking如何通过修改字节码让插件生效](https://blog.csdn.net/qq_18515155/article/details/114454166)。

工作中我们也可以尝试借鉴这种思想来扩展我们的业务服务。

最后让我们一起喊出：**动态代理，yyds！**

---
相关代码请参考：[https://github.com/xujian01/blogcode/tree/master/src/main/java/mybatisplugin](https://github.com/xujian01/blogcode/tree/master/src/main/java/mybatisplugin)