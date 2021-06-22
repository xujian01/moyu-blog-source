---
title: Gson序列化LinkedHashMap.Entry失败的探索
date: 2021-03-26 10:33:29
tags: Gson
---
Gson对特定对象序列化失败的探索过程。
<!-- more -->
## 问题重现
示例代码如下：
```java
Map<String, Object> map = new LinkedHashMap<>();
        map.put("name", "xujian");
        map.put("age", 25);
        Gson gson = new Gson();
        for (Map.Entry<String, Object> entry : map.entrySet()) {
   System.out.println(gson.toJson(entry));
        }
```
打印结果如下：
![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-yoafvffX-1612510784802)(media/16122669851328/16122674360488.jpg)\]](https://img-blog.csdnimg.cn/20210205154200342.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)


最初看到这个问题是一个同学问我，问题是这样的：
*为什么用LinkedHashMap就会报`StackOverflowError`，但是使用普通的HashMap就没问题？*

因为我前不久刚好写了一篇关于Bean Copy的博客（感兴趣可以看[Bean Copy也就这么点事了！](https://blog.csdn.net/qq_18515155/article/details/111414852)）。
里面讲到了Bean Copy的原理和方案，里面提到了使用json序列化和反序列化来做“深拷贝”，而“深拷贝”往往就是通过递归处理对象之间的引用来实现。

所以看到这个问题，第一反应就是Gson在序列化的时候是不是因为有“循环引用”的存在，导致在递归处理对象引用的时候出现了“死递归”导致堆栈溢出。

## 原因探究
带着上面的猜测去网上搜寻答案，搜到的基本都是说Gson本身不解决“循环引用”，而FastJson、Jackson默认会处理“循环引用”。虽然没有完全证实我的猜测，但至少证明`LinkedHashMap.Entry`**应该是存在着循环引用**。

为了验证这个结论，通过源码（基于JDK1.8）分别来看看HashMap和LinkedHashMap的Entry有啥不同。

HashMap:
```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    ...
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
        ...
    }
    transient Node<K,V>[] table;
    ...
}
```
> 从上面代码中可以看到，Node对象持有一个指向下一个Node节点的指针。

LinkedHashMap:
```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    ...
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
    transient LinkedHashMap.Entry<K,V> head;
    transient LinkedHashMap.Entry<K,V> tail;
    ...
}
```
> 从上面代码可以看出Entry里面持有两个指针，一个指向前面的Entry，一个指向后面的Entry。

LinkedHashMap比HashMap多一个指针，而恰恰就是这个指向前一个Entry的指针，让其形成了“循环”结构。

依据这个结论是不是可以大胆猜测使用Gson对`TreeMap.Entry`序列化也会报“堆栈溢出”错误。

示例如下：
```java
Map<String, Object> map = new TreeMap<>();
        map.put("name", "xujian");
        map.put("age", 25);
        Gson gson = new Gson();
        for (Map.Entry<String, Object> entry : map.entrySet()) {
            System.out.println(gson.toJson(entry));
        }
```
结果如下：
![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-pG1PBZV1-1612510784803)(media/16122669851328/16122695573490.jpg)\]](https://img-blog.csdnimg.cn/20210205154246844.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)

证明我的猜测是对的。因为TreeMap和LinkedHashMap一样的，他的Entry里面除了指向左右子节点的两个指针，还有一个指向父节点的指针，也正是因为这个指向父节点的指针，使其形成了”循环“结构。
TreeMap：
```java
public class TreeMap<K,V>
    extends AbstractMap<K,V>
    implements NavigableMap<K,V>, Cloneable, java.io.Serializable
{
    ...
    static final class Entry<K,V>   implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left;
        Entry<K,V> right;
        Entry<K,V> parent;
        boolean color = BLACK;
        ...
    }
    ...
}
```
根据以上探索过程，可以得出下面的结论：
1. `LinkedHashMap.Entry`因为持有对前一个Entry的指针，造成“循环引用”；`TreeMap.Entry`因为持有对父节点的指针，造成“循环引用”；
2. Gson本身不支持对象“循环引用”的情况；
3. Gson递归序列化时出现“死递归”，从而“堆栈溢出”；

## 验证结论
上面的主要内容是根据现象探究原理得出结论。
现在用上面得出的结论来套到一个自己的实例里面，看能否正确解释。
仿照`LinkedHashMap.Entry`定义一个自己的双向链表结构：
```java
public class MyOneWayEntry {
    int value;
    MyOneWayEntry next;
    MyOneWayEntry prev;

    public MyOneWayEntry(int value) {
        this.value = value;
    }

    public void setValue(int value) {
        this.value = value;
    }

    public void setNext(MyOneWayEntry next) {
        this.next = next;
    }

    public void setPrev(MyOneWayEntry prev) {
        this.prev = prev;
    }
}
```
> 这里出现了前后两个指针。

测试代码如下：
```java
MyOneWayEntry myOneWayEntry2 = new MyOneWayEntry(2);
        MyOneWayEntry myOneWayEntry1 = new MyOneWayEntry(1);
        //让myOneWayEntry2的prev指针指向myOneWayEntry1
        myOneWayEntry2.setPrev(myOneWayEntry1);
        //让myOneWayEntry1的next指针指向myOneWayEntry2
        myOneWayEntry1.setNext(myOneWayEntry2);
        Gson gson = new Gson();
        System.out.println(gson.toJson(myOneWayEntry1));
```
结果也出现了和打印`LinkedHashMap.Entry`同样的错误:
![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-pBmRCnkt-1612510784805)(media/16122669851328/16125075414997.jpg)\]](https://img-blog.csdnimg.cn/20210205154231945.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)

但是如果把上面的prev和next去掉其中一个，则会正常打印json序列化字符串，感兴趣的可以自己试试。

类似的，如果把prev和next指针其中一个用`transient`修饰，如下所示：
```java
MyOneWayEntry next;
transient MyOneWayEntry prev;
```
也会正常序列化，只不过序列化的结果里面没有了prev字段：
```
{"value":1,"next":{"value":2}}
```
> 这样的结果完全可以用上面的结论来解释了。

## 使用Gson序列化LinkedHashMap本身为什么没问题
事情到上面还没有结束：既然Gson序列化`LinkedHashMap.Entry`会因为循环引用报错堆栈溢出，那LinkedHashMap本身也是持有`LinkedHashMap.Entry`的引用，为什么Gson序列化的时候不会报错呢？

如果你仔细看看LinkedHashMap的源码，你会发现LinkedHashMap对`LinkedHashMap.Entry`的引用使用`transient`修饰了，而`transient`会告诉Gson，它修饰字段**不参与序列化**。

可以自己定义一个简单的LinkedHashMap结构，但是将其对Entry的引用不用`transient`修饰：
```java
public class MyLinkedHashMap {
    MyLinkedHashMap.MyDoubleLinkEntry head;

    public void setHead(MyDoubleLinkEntry head) {
        this.head = head;
    }

    static class MyDoubleLinkEntry {
        MyDoubleLinkEntry before;
        MyDoubleLinkEntry after;
        int value;

        public MyDoubleLinkEntry(int value) {
            this.value = value;
        }

        public void setBefore(MyDoubleLinkEntry before) {
            this.before = before;
        }

        public void setAfter(MyDoubleLinkEntry after) {
            this.after = after;
        }

        public void setValue(int value) {
            this.value = value;
        }
    }
}
```
测试代码如下：
```java
MyLinkedHashMap myLinkedHashMap = new MyLinkedHashMap();
        MyLinkedHashMap.MyDoubleLinkEntry first = new MyLinkedHashMap.MyDoubleLinkEntry(1);
        MyLinkedHashMap.MyDoubleLinkEntry second = new MyLinkedHashMap.MyDoubleLinkEntry(2);
        first.setAfter(second);
        second.setBefore(first);
        myLinkedHashMap.setHead(first);
        Gson gson = new Gson();
        System.out.println(gson.toJson(myLinkedHashMap));
```
结果是序列化的时候也报错“StackOverflowError”。
如果加上`transient`修饰`MyLinkedHashMap.MyDoubleLinkEntry head`字段，则会正常序列化。
## 其他拓展
每种json序列化框架都有自己的忽略某个字段的方式，如用`transient`修饰，FastJson的`@JSONField(serialize = false,deserialize = false)`，Jackson的`@JsonIgnore`,Gson的`@Expose(serialize = false, deserialize = false)`配合
```java
GsonBuilder builder = new GsonBuilder();  
builder.excludeFieldsWithoutExposeAnnotation();  
Gson gson = builder.create();
```
当然还有其他各种方式，感兴趣可以自己查找整理一下。
## 总结
1. Gson默认不支持存在“循环引用”的对象的序列化；
2. `LinkedHashMap.Entry`中存在before, after双指针，形成了“循环引用”结构，Gson序列化会报错“StackOverflowError”；
3. LinkedHashMap对`LinkedHashMap.Entry`的引用使用了`transient`修饰，所以序列化才不会报错；
4. 这里顺便提一下，对于Map.Entry的序列化，FastJson是用`Entry.getKey()`和`Entry.getValue()`获取数据的；而Gson是用反射`Field.get()`来获取数据的；

---
你可以在这里获取相关代码：[https://github.com/xujian01/blogcode/tree/master/src/main/java/gson](https://github.com/xujian01/blogcode/tree/master/src/main/java/gson)