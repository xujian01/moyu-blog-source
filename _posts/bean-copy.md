---
title: Bean Copy也就这么点事了！
date: 2020-08-10 10:33:29
tags: Java
---
工作中经常需要用到对象的Copy，这篇文章系统对对象Copy做一个系统的总结。
<!-- more -->
# 什么是深拷贝和浅拷贝
先来看一张图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020122011033046.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)
我们都知道对象的引用分配在栈上，对象的内存分配在堆上。
如上图所示，对象objB是对象objA的一个拷贝，objB只是在栈上重新分配了一个引用，但是实际指向的内存和objA是同一个，这就是**浅拷贝**；相反，如果objB除了在栈上分配了新的引用，并且在堆上也分配了新的内存空间，这就是**深拷贝**。

> 浅拷贝就是复制对象的引用，修改复制的对象会影响原来的对象；深拷贝就是新建一个和原对象“一模一样”的对象，但是他们在堆上占用不同的内存。

*题外话：和深拷贝（Deep Copy）、浅拷贝（Shallow Copy），有相似概念的还有深堆（Deep Heap）、浅堆（Shallow Heap），感兴趣的同学可以自己查找资料。*
## 浅拷贝
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201220140829162.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)
1. 对于数据类型是**基本数据类型**的成员变量，浅拷贝会直接将该属性值复制一份给新的对象。因为是两份不同的数据，所以对其中一个对象的该成员变量值进行修改，不会影响另一个对象拷贝得到的数据。
2. 对于数据类型是**引用数据类型**的成员变量，比如说成员变量是某个数组、某个类的对象等，那么浅拷贝会将该成员变量的引用值（内存地址）复制一份给新的对象。因为实际上两个对象的该成员变量都指向同一个实例。在这种情况下，在一个对象中修改该成员变量会影响到另一个对象的该成员变量值。
3. String类型非常特殊，首先，String类型**属于引用数据类型**，不属于基本数据类型，但是String类型的数据是存放在常量池中的，也就是无法修改的！也就是说，当修改String类型成员变量后，并不是修改了这个数据的值，而是把这个数据的**引用指向了一个新的字符串常量**。
### 实现方式
#### 使用拷贝构造方法
```java
public static void main (String[] args) {
	Address addr = new Address("北京")
	User userA = new User(10,addr);
	System.out.println(userA.getAge());
	//浅拷贝
	User userC = new User(userA);
	userC.setAge(30);
	System.out.println(userA.getAge());
	System.out.println(userC.getAge());
	System.out.println(userA.getAddr() == userCopy.getAddr());
}

@Getter
@Setter
public class User {
	private int age;
	private Address addr;
	public User(int age,Address addr) {
		this.age = age;
		this.addr = addr;
	}
	//复制构造方法，模拟浅拷贝
	public User(User source) {
        this.age = source.age;
        this.addr = source.addr;
    }
}
public class Address {
	private String addr;
	public Address(String addr) {
		this.addr = addr;
	}
}
```
输出结果：

```html
10
10
20
false
```
#### 使用Cloneable
1. 实现`Cloneable`接口；
2. 重写Object类的`clone()`方法；

`Address.java`
```java
/**
 * @author xujian
 * @date 2020-12-20 14:52
 **/
public class Address {
    private String addr;

    public Address(String addr) {
        this.addr = addr;
    }

    public String getAddr() {
        return addr;
    }

    public void setAddr(String addr) {
        this.addr = addr;
    }

    @Override
    public String toString() {
        return "Address{" +
                "addr='" + addr + '\'' +
                '}';
    }
}
```
`User.java`

```java
/**
 * @author xujian
 * @date 2020-12-20 14:52
 **/
public class User implements Cloneable{
    private int age;
    private Address addr;

    public User(int age, Address addr) {
        this.age = age;
        this.addr = addr;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public Address getAddr() {
        return addr;
    }

    public void setAddr(Address addr) {
        this.addr = addr;
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }

    @Override
    public String toString() {
        return "User{" +
                "age=" + age +
                ", addr=" + addr +
                '}';
    }
}
```
测试

```java
/**
 * @author xujian
 * @date 2020-12-20 15:08
 **/
public class Demo {
    public static void main(String[] args) throws CloneNotSupportedException {
        Address addr = new Address("北京市");
        User userA = new User(10,addr);
        User userCopy = (User) userA.clone();
        System.out.println(userA);
        System.out.println(userCopy);
        //判断userA和userCopy是否是同一个对象
        System.out.println(userA == userCopy);
        userCopy.setAge(20);
        addr.setAddr("西安市");
        System.out.println(userA);
        System.out.println(userCopy);
        //判断userA的成员变量addr和userCopy的成员变量addr是否是同一个对象
        System.out.println(userA.getAddr() == userCopy.getAddr());
    }
}
```
输出

```javascript
User{age=10, addr=Address{addr='北京市'}}
User{age=10, addr=Address{addr='北京市'}}
false
User{age=10, addr=Address{addr='西安市'}}
User{age=20, addr=Address{addr='西安市'}}
true
```
1. 基本类型的成员变量原始对象和拷贝对象互不影响；
2. 可以看到clone()方法拷贝的userCopy对象和原对象的**内容是一样的**。然而他们并**不是同一个对象**；
3. 原始对象和拷贝的象他们的引用类型成员变量`Address::addr`是同一个对象;
4. Object::clone()本身是个浅拷贝实现;

⚠️注意
1. 使用`clone()`方法时必须实现Cloneable接口，否则会报错`CloneNotSupportedException`，其实Cloneable接口没有定义任何方法，它和`Serializable`接口一样，都是起到一个标识作用;
2. `clone()`是一个native方法，所以在浅拷贝上其效率还是很好的;
3. `clone()`三原则：
① 对任何的对象x，都有x.clone() !=x//克隆对象与原对象不是同一个对象；
② 对任何的对象x，都有x.clone().getClass()==x.getClass()//克隆对象与原对象的类型一样；
③ 如果对象x的equals()方法定义恰当，那么x.clone().equals(x)应该成立；
#### 使用Setter方法
通过对象内部提供的`Setter`方法简单且直接的进行对象的拷贝：
```java
public static User copyBySetter(User source){
        User user = new User();
        user.setAge(source.getAge());
        user.setAddr(source.getAddr());
        return user;
    }
```
## 深拷贝
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020122014085249.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)
1. 不仅要复制对象的所有基本数据类型的成员变量值，还要为所有引用数据类型的成员变量申请存储空间，并复制每个引用数据类型成员变量所引用的对象，一直**递归**下去直到没有引用类型的成员变量。也就是说，对象进行深拷贝要对整个对象图进行拷贝；
2. 深拷贝对引用数据类型的成员变量的对象图中所有的对象都开辟了内存空间；而浅拷贝只是传递地址指向，新的对象并没有对引用数据类型创建内存空间；
### 实现方式
#### 使用Cloneable
在上面使用`Cloneable`实现浅拷贝的基础上，对`clone`方法进行改造：
`City.java`

```java
/**
 * @author xujian
 * @date 2020-12-26 12:30
 **/
public class City implements Cloneable, Serializable{
    private String name;

    public City() {

    }

    public City(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }

    @Override
    public String toString() {
        return "City{" +
                "name='" + name + '\'' +
                '}';
    }
}
```
`Address.java`

```java
/**
 * @author xujian
 * @date 2020-12-20 14:52
 **/
public class Address implements Cloneable, Serializable{
    private String addr;
    private City city;

    public Address() {

    }

    public Address(String addr) {
        this.addr = addr;
    }

    public String getAddr() {
        return addr;
    }

    public void setAddr(String addr) {
        this.addr = addr;
    }

    public void setCity(City city) {
        this.city = city;
    }

    public City getCity() {
        return city;
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        Address address = (Address) super.clone();
        City city = (City) address.getCity().clone();
        address.setCity(city);
        return address;
    }

    @Override
    public String toString() {
        return "Address{" +
                "addr='" + addr + '\'' +
                ", city=" + city +
                '}';
    }
}
```
`User.java`

```java
/**
 * @author xujian
 * @date 2020-12-20 14:52
 **/
public class User implements Cloneable, Serializable{
    private int age;
    private Address addr;

    public User() {

    }
    public User(int age, Address addr) {
        this.age = age;
        this.addr = addr;
    }

    public User(User source) {
        this.age = source.age;
        this.addr = source.addr;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public Address getAddr() {
        return addr;
    }

    public void setAddr(Address addr) {
        this.addr = addr;
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        User user = (User) super.clone();
        Address address = (Address) user.getAddr().clone();
        user.setAddr(address);
        return user;
    }

    @Override
    public String toString() {
        return "User{" +
                "age=" + age +
                ", addr=" + addr +
                '}';
    }
}
```

```java
public static void main(String[] args) throws CloneNotSupportedException {
        User source = new User();
        source.setAge(70);
        Address address = new Address();
        address.setAddr("陕西省");
        City city = new City("渭南市");
        address.setCity(city);
        source.setAddr(address);

        User copyUser = (User) source.clone();
        System.out.println(source);
        System.out.println(copyUser);
        System.out.println(source == copyUser);
    }
```
输出结果如下：

```javascript
User{age=70, addr=Address{addr='陕西省', city=City{name='渭南市'}}}
User{age=70, addr=Address{addr='陕西省', city=City{name='渭南市'}}}
false
```

> 这种方式在`clone()`方法的浅拷贝基础上，根据引用类型成员变量使用层层调用，来实现深拷贝，也是有点繁琐的。

#### 序列化方式

> 这种方式在我们日常开发工作中非常常见，避免了手动实现深拷贝中“递归”的操作，所以看上去会比上面的方式简单一点。

##### Json序列化/反序列化
可以想一想，SpringMVC中的Controller在传递对象参数和返回对象数据时的序列化和反序列化。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201226161120388.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)
> 从Json字符串反序列化来的对象都是一个新的对象。

依旧是上面的例子：
```java
public static void main(String[] args) throws CloneNotSupportedException {
        User source = new User();
        source.setAge(70);
        Address address = new Address();
        address.setAddr("陕西省");
        City city = new City("渭南市");
        address.setCity(city);
        source.setAddr(address);

		//序列化成json字符串
        String userJsonStr = JSON.toJSONString(source);
        //反序列化成对象
        User copyUser = JSON.parseObject(userJsonStr,User.class);

        System.out.println(source);
        System.out.println(copyUser);
        System.out.println(source == copyUser);
    }
```
输出结果：

```javascript
User{age=70, addr=Address{addr='陕西省', city=City{name='渭南市'}}}
User{age=70, addr=Address{addr='陕西省', city=City{name='渭南市'}}}
false
```
> 可见反序列化生成的对象是一个新的对象。
##### JDK序列化/反序列化
同样还有JDK自带的序列化方式。
依旧是上面的例子：

```java
public static void main(String[] args) throws Exception {
        User source = new User();
        source.setAge(70);
        Address address = new Address();
        address.setAddr("陕西省");
        City city = new City("渭南市");
        address.setCity(city);
        source.setAddr(address);
        
        User copyUser = jdkDeepCopy(source);

        System.out.println(source);
        System.out.println(copyUser);
        System.out.println(source == copyUser);
    }
    /**
     * jdk序列化/反序列化
     * @param source
     * @return
     * @throws Exception
     */
    public static User jdkDeepCopy(User source) throws Exception {
        //将对象写到流里
        ByteArrayOutputStream bo=new ByteArrayOutputStream();
        ObjectOutputStream oo=new ObjectOutputStream(bo);
        oo.writeObject(source);
        //从流里读出来
        ByteArrayInputStream bi=new ByteArrayInputStream(bo.toByteArray());
        ObjectInputStream oi=new ObjectInputStream(bi);
        return (User) oi.readObject();
    }
```
输出结果：

```javascript
User{age=70, addr=Address{addr='陕西省', city=City{name='渭南市'}}}
User{age=70, addr=Address{addr='陕西省', city=City{name='渭南市'}}}
false
```
#### 手动使用Setter层层设置属性
还是基于上面的例子，增加一个手动使用Setter进行copy的方法：
```java
public static void main(String[] args) {
        User source = new User();
        source.setAge(70);
        Address address = new Address();
        address.setAddr("陕西省");
        City city = new City("渭南市");
        address.setCity(city);
        source.setAddr(address);

        User copyUser = getDeepCopyInstance(source);
        System.out.println(source);
        System.out.println(copyUser);
        System.out.println(source == copyUser);
    }

    /**
     * 得到指定对象的一个深拷贝对象
     * @return
     */
    public static User getDeepCopyInstance(User source) {
        User copyUser = new User();
        copyUser.setAge(source.getAge());
        Address address = new Address();
        address.setAddr(source.getAddr().getAddr());
        City city = new City();
        city.setName(source.getAddr().getCity().getName());
        address.setCity(city);
        copyUser.setAddr(address);
        return copyUser;
    }
```
输出结果：

```javascript
User{age=70, addr=Address{addr='陕西省', city=City{name='渭南市'}}}
User{age=70, addr=Address{addr='陕西省', city=City{name='渭南市'}}}
false
```
> 这个方法虽然可以达到效果，但是编码繁琐，并且不够通用，试想引用链很长的时候编码工作量得多恐怖。

其他方式还可以借助**反射直接进行属性赋值，内省机制（依赖于反射）通过调用Setter方法，构造函数**等通过层层递归来进行对象的深拷贝。
# Bean拷贝工具
## Apache的BeanUtils
位于`org.apache.commons.beanutils`包下。通过上面提到的**内省机制**调用`Setter`方法实现。默认实现**浅拷贝**，想要实现深拷贝，则需要提供自定义的`Converter`。
使用示例：
```java
public class BeanUtilsDemo {

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        UserDO userDO = DataUtil.createData();
        log.info("拷贝前,userDO:{}", userDO);
        UserDTO userDTO = new UserDTO();
        BeanUtils.copyProperties(userDO,userDTO);
        log.info("拷贝后,userDO:{}", userDO);
    }
}
```
输出结果：
```javascript
18:12:11.734 [main] INFO cn.van.parameter.bean.copy.demo.BeanUtilsDemo - 拷贝前,userDO:UserDO(id=1, userName=Van, gmtBroth=2019-11-02T18:12:11.730, balance=100)
18:12:11.917 [main] INFO cn.van.parameter.bean.copy.demo.BeanUtilsDemo - 拷贝后,userDO:UserDO(id=1, userName=Van, gmtBroth=2019-11-02T18:12:11.730, balance=100)
```
> 稳定性与效率都不行，不推荐使用。
## Apache的PropertyUtils
位于`org.apache.commons.beanutils`包下。
`PropertyUtils`的`copyProperties()`方法几乎与`BeanUtils.copyProperties()`相同，主要的区别在于`BeanUtils.copyProperties()`提供**类型转换**功能，即发现两个JavaBean的同名属性为不同类型时，在支持的数据类型范围内进行转换，PropertyUtils不支持这个功能，所以说BeanUtils使用更普遍一点，犯错的风险更低一点，默认实现**浅拷贝**。
## Apache的SerializationUtils
位于`org.apache.commons.lang3`包下。
`SerializationUtils.clone(T)`，T对象需要实现 Serializable 接口，基于JDK序列化实现了**深克隆**。
> 深拷贝可以使用。
## Spring的BeanUtils
位于`org.springframework.beans`包下。
其`copyProperties`方法实现原理和`Apache BeanUtils.copyProperties`原理类似，默认实现**浅拷贝**，区别在于对`PropertyDescriptor`（内省机制相关）的处理结果做了缓存来提升性能。
## Spring的BeanCopier
位于`org.springframework.cglib.beans`包下，默认实现**浅拷贝**。
BeanCopier支持两种方式:
1. 不使用Converter的方式，仅对两个bean间属性名和类型完全相同的变量进行拷贝;
2. 引入Converter，可以对某些特定属性值进行特殊操作，但性能会大大降低。

通过操作字节码生成类，来实现原本需要通过反射或者一堆代码才能实现的逻辑，内部还是通过直接调用`Setter`方法来实现。
> 使用了字节码操作取代反射，使用缓存以及通过`Setter`方法设置属性值使其性能非常优秀，推荐使用。

**Tips**：
*实际使用时，可以把创建过的BeanCopier实例放到缓存中，下次可以直接获取，提升性能。*
## MapStruct
默认实现**浅拷贝**，通过注解编译时自动生成拷贝的代码，基于直接调用`Setter`方法实现。
> 性能还不错，主要是使用方便，加上注解就可以用了。
# 总结
1. 浅拷贝：是对原对象的属性值进行精准复制，那么对如果原对象的属性值是基本类型那就是值的引用，所以浅拷贝后修改基本类型不会修改到原对象的，如果原对象属性值是引用类型，那么就是对引用类型属性值的栈内存的复制，所以修改引用类型属性值的时候回修改到原对象。
2. 深拷贝：在浅拷贝的基础上，对引用类型开辟新的堆内存进行层层拷贝。
3. 上文中提到的深堆，浅堆，内省机制有兴趣的可以自行了解。

上文示例代码请参考：[浅拷贝](https://github.com/xujian01/blogcode/tree/master/src/main/java/shallowcopy)、[深拷贝](https://github.com/xujian01/blogcode/tree/master/src/main/java/deepcopy)。
# 观众老爷求点赞