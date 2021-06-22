---
title: 探索Spring循环依赖的细节
date: 2021-05-10 10:33:29
tags: Spring
---
之前在[Spring学习笔记，挺全的！](https://blog.csdn.net/qq_18515155/article/details/117334765)这个博客中提到了Spring对循环依赖的解决原理，这篇博客对一些细节做个补充。
<!-- more -->
## 背景回顾

```java
@Service
public class LagouBean {
  @Autowired
  private ItBean itBean;
  
  public ItBean getItBean() {
        return itBean;
    }
}

@Service
public class ItBean {
  @Autowired
  private LagouBean lagouBean;
}
```

> LagouBean被SpringAOP增强（代码省略）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/202105271824271.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)

之前只是扔出了这张图片，现在来补充一下文字说明。

**循环依赖解决步骤：**

1）实例化lagouBean，然后将其放入三级缓存singletonFactories，key是beanName，value是一个对象工厂ObjectFactory

```java
new ObjectFactory<Object>() {
				@Override
				public Object getObject() throws BeansException {
					return getEarlyBeanReference(beanName, mbd, bean);
				}
```

2）对lagouBean进行属性注入，发现其依赖了itBean

3）实例化itBean，然后将其放入三级缓存，key是beanName，value是一个对象工厂ObjectFactory

```java
new ObjectFactory<Object>() {
				@Override
				public Object getObject() throws BeansException {
					return getEarlyBeanReference(beanName, mbd, bean);
				}
```

4）对itBean进行属性注入，发现其依赖于lagouBean

5）通过依赖解析调用`org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(String beanName, boolean allowEarlyReference)`，beanName此时传递“lagouBean”，allowEarlyReference传递true。

从三级缓存singletonFactories拿到lagouBean的对象工厂ObjectFactory，调用其getObject方法，进而调用getEarlyBeanReference。

6）在getEarlyBeanReference里面会遍历所有的BeanPostProcessor找到特定类型的BeanPostProcessor（SmartInstantiationAwareBeanPostProcessor），调用其getEarlyBeanReference方法，会对发生【循环依赖】的bean即这里的lagouBean暴露一个“早期引用”，如果该bean需要进行SpringAOP增强，那么暴露的是一个该bean即这里的lagouBean的**代理类**的引用，同时将这个代理类对象放到二级缓存earlySingletonObjects中

7）将暴露的lagouBean的代理类注入到itBean，itBean完成创建以后会从三级缓存singletonFactories中移除，并将其放到一级缓存singletonObjects中

8）将创建完成的itBean注入到lagouBean，然后进行后置处理逻辑（BeanPostProcessor等逻辑）并返回处理完以后的lagouBean，**如果处理完以后的lagouBean和处理之前的lagouBean一样（是同一个对象）**，从二级缓存earlySingletonObjects获取lagouBean对应的值，经过步骤6，将会获取到lagouBean的代理类对象，然后把该代理类对象返回

9）将步骤8返回的lagouBean代理类对象放到一级缓存singletonObjects中（key为“lagouBean”，value为lagouBean代理类对象）

此时lagouBean和itBean的引用关系如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021060919183360.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)
## 细节探索

注意到上面的步骤8，对应的部分源码如下：

`org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.java`

```java
//bean是刚刚实例化的对象
Object exposedObject = bean;
//1、属性注入
populateBean(beanName, mbd, instanceWrapper);
if (exposedObject != null) {
  //2、对bean进行BeanPostProcessor等处理，SpringAOP动态代理也在此处生成，返回一个代理对象
		exposedObject = initializeBean(beanName, exposedObject, mbd);
}
//3、从二级缓存获取早期引用即lagouBean的代理对象
Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
        //4、判断是否在initializeBean过程中生成代理对象并返回
				if (exposedObject == bean) {
          //5、将exposedObject指向lagouBean的代理对象
					exposedObject = earlySingletonReference;
				}
        //6、如果发现有已经创建完成的bean依赖了beanName对应的bean，并且这个依赖的bean和exposedObject不相同，就报循环依赖异常
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
//7、返回
return exposedObject;
```

应该会走上面的5处的代码，就说明**initializeBean方法执行完成返回的bean和参数传入的bean是一样的！**

### 为什么一样

根据示例代码，对LagouBean是要进行SpringAOP增强的，那应该会在initializeBean中执行相应的BeanPostProcessor生成代理对象才是啊？

继续分析源码，lagoBean在进行属性注入以后会进入initializeBean，然后找到SpringAOP相关的BeanPostProcessor即AbstractAutoProxyCreator，其postProcessAfterInitialization方法如下：

`org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator.java`

```java
@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
      //重点关注，earlyProxyReferences不包含当前beanName的时候才去走生成代理类的逻辑
			if (!this.earlyProxyReferences.contains(cacheKey)) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
```

可以看到在earlyProxyReferences不包含当前beanName的时候才去走生成代理类的逻辑，否则直接原样返回。

还记得循环依赖解决的步骤6吗？部分源码如下：

`org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.java`

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
          //获取代理对象
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
            //放入二级缓存
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}
```

其中`ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName)`会走到

`org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.java`

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
		Object exposedObject = bean;
		if (bean != null && !mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
					SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
          //获取代理对象
					exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
					if (exposedObject == null) {
						return null;
					}
				}
			}
		}
		return exposedObject;
	}
```

其中`exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);`会走到

`org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator.java`

```java
public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
	//1、根据beanName信息获取cacheKey	
  Object cacheKey = getCacheKey(bean.getClass(), beanName);
		if (!this.earlyProxyReferences.contains(cacheKey)) {
      //2、将cacheKey放到earlyProxyReferences
			this.earlyProxyReferences.add(cacheKey);
		}
		return wrapIfNecessary(bean, beanName, cacheKey);
	}

public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (bean != null) {
      //3、根据beanName信息获取cacheKey
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
      //4、earlyProxyReferences不包含当前beanName的时候才去走生成代理类的逻辑
			if (!this.earlyProxyReferences.contains(cacheKey)) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
```

可以看到在这里将参数传递的bean对应的cacheKey放入了earlyProxyReferences，

代码2和代码4正好呼应上，整体过程如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210608202238660.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)


话又说回来了，上面分析的是initializeBean方法执行完成返回的bean和参数传入的bean为什么是一样的，那什么情况下不一样，从而抛出循环依赖异常呢？

### 什么情况下会不一样

还是LagouBean和ItBean的例子，只不过对LagouBean不使用Aspect来增强，而是通过自定义BeanPostProcessor来生成代理对象，代码如下：

```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object o, String s) throws BeansException {
        return o;
    }

    @Override
    public Object postProcessAfterInitialization(Object o, String s) throws BeansException {
        if ("lagouBean".equals(s)) {
            return Proxy.newProxyInstance(o.getClass().getClassLoader(),
                    o.getClass().getInterfaces(),
                    (proxy, method, args) -> {
                        System.out.println("----开始");
                        Object o1 = method.invoke(o,args);
                        System.out.println("----结束");
                        return o1;
                    });
        } else {
            return o;
        }
    }
}
```

重新启动spring发现报错

```
Caused by: org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'lagouBean': Bean with name 'lagouBean' has been injected into other beans [itBean] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.

```

继续跟踪源码发现是因为initializeBean方法执行完成返回的bean和参数传入的bean不一样了。

因为在执行了initializeBean之后返回了由自定义MyBeanPostProcessor生成的lagouBean的代理对象，此时lagouBean最终对应的是一个lagouBean的代理对象，而当初itBean注入lagouBean的时候注入的是lagouBean的原始对象，这就产生了矛盾，继续往下执行就会抛出上面的异常。

### lagouBean.getItBean()问题

再来回顾一下上面循环依赖解决之后lagouBean和itBean的引用关系

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210609191857246.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)

可以看到其实lagouBean对应的是一个lagouBean的代理对象，而这个代理对象中的itBean属性是null！

那lagouBean.getItBean()岂不是获取到的是个null？经过测试事实并不会返回null，而是正常返回了itBean的对象。

那继续来看看源码到底发生了什么

首先给lagouBean.getItBean()这行代码打断点发现进入的并不是LagouBean本身的getItBean()方法，而是进入了`org.springframework.aop.framework.CglibAopProxy.DynamicAdvisedInterceptor.intercept`方法，从前面我们知道lagouBean的代理对象是通过cglib产生的，对cglib感兴趣的可以参考[Cglib原理解析](https://blog.csdn.net/yhl_jxy/article/details/80633194)。

而`org.springframework.aop.framework.CglibAopProxy.DynamicAdvisedInterceptor`正好实现了MethodInterceptor接口，所以对于cglib代理类方法的调用就会进入该类的intercept方法，部分源码如下：

`org.springframework.aop.framework.CglibAopProxy.java`

```java
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
  ...
  retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
	...
}
```

最终会进入到

`org.springframework.aop.framework.ReflectiveMethodInvocation.java`

```java
public Object proceed() throws Throwable {
		//	We start with an index of -1 and increment early.
  //遍历interceptorsAndDynamicMethodMatchers
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
      //反射调用被代理类的原始方法
			return invokeJoinpoint();
		}

		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		...
      //执行增强逻辑
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
  	...
}
```

可以看到依次执行interceptorsAndDynamicMethodMatchers中保存的拦截器的invoke方法来完成增强逻辑（这种设计思路可以参考[实现一个增强版的JDK动态代理](https://blog.csdn.net/qq_18515155/article/details/117334651)来简单理解），执行完以后会通过反射去调用被代理的目标类的原始的目标方法，返回上面lagouBean和itBean依赖关系中target所依赖的itBean对象。

---

相关代码请参考：[https://gitee.com/xujian01/blogcode/tree/master/springcode/src/main/java/com/jarry/circledepency](https://gitee.com/xujian01/blogcode/tree/master/springcode/src/main/java/com/jarry/circledepency)