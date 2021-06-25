---
title: 使用三个线程，按顺序打印X，Y，Z，连续打印10次
date: 2021-06-25 19:56:29
tags: 多线程
---
题目描述：
使用三个线程，按顺序打印X，Y，Z，连续打印10次
<!-- more -->
```java
/**
 * 题目描述：使用三个线程，按顺序打印X，Y，Z，连续打印10次。
 * @author xujian
 * 2021-06-25 13:38
 **/
public class PrintXYZ {
    //定义CountDownLatch，起到线程通知的作用
    private static CountDownLatch cd1 = new CountDownLatch(1);
    private static CountDownLatch cd2 = new CountDownLatch(1);
    private static CountDownLatch cd3 = new CountDownLatch(1);

    public static void main(String[] args) {
        Thread x = new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    //线程启动之后等待，直到cd1的门闩变为0
                    cd1.await();
                    System.out.println(Thread.currentThread().getName()+":X");
                    //将cd2的门闩减为0，使cd2可以从等待返回
                    cd2.countDown();
                    //新建一个CountDownLatch用于下一次循环
                    cd1 = new CountDownLatch(1);
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"print-x");

        Thread y = new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    cd2.await();
                    System.out.println(Thread.currentThread().getName()+":Y");
                    cd3.countDown();
                    cd2 = new CountDownLatch(1);
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"print-y");

        Thread z = new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    cd3.await();
                    System.out.println(Thread.currentThread().getName()+":Z");
                    System.out.println("-----------");
                    cd1.countDown();
                    cd3 = new CountDownLatch(1);
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"print-z");
        x.start();
        y.start();
        z.start();
        //三个线程启动完都会等待各自的门闩
        //cd1门闩先减为0，意味着线程print-x会先从等待中返回，从而先打印X
        cd1.countDown();
    }
}
```
效果展示

```shell
print-x:X
print-y:Y
print-z:Z
-----------
print-x:X
print-y:Y
print-z:Z
-----------
print-x:X
print-y:Y
print-z:Z
-----------
print-x:X
print-y:Y
print-z:Z
-----------
print-x:X
print-y:Y
print-z:Z
-----------
print-x:X
print-y:Y
print-z:Z
-----------
print-x:X
print-y:Y
print-z:Z
-----------
print-x:X
print-y:Y
print-z:Z
-----------
print-x:X
print-y:Y
print-z:Z
-----------
print-x:X
print-y:Y
print-z:Z
-----------
```
> 这是我自己想的办法，能实现效果。如果大家有更好的办法欢迎指导～

相关代码请参考：[https://gitee.com/xujian01/blogcode/tree/master/src/main/java/thread](https://gitee.com/xujian01/blogcode/tree/master/src/main/java/thread)