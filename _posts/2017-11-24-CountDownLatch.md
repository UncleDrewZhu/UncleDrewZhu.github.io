---
layout:     post
title:      CountDownLatch 简述
subtitle:   关于 CountDownLatch 的一些基础解释 
date:       2017-11-24
author:     uncledrew
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - 线程
    - CountDownLatch
---

> 纸上得来终觉浅，绝知此事要躬行。
>
> [我的博客](http://uncledrew.405go.cn/)


# CountDownLatch
CountDownLatch 这个类能够使一个线程等待其他线程完成各自的工作后再执行。

CountDownLatch 是通过一个计数器来实现的，计数器的初始值为线程的数量。每当一个线程完成了自己的任务后，计数器的值就会减1。
当计数器值到达0时，它表示所有的线程已经完成了任务，然后在闭锁上等待的线程就可以恢复执行任务。

CountDownLatch的伪代码如下所示：
- Main thread start
- Create CountDownLatch for N threads
- Create and start N threads
- Main thread wait on latch
- N threads completes there tasks are returns
- Main thread resume execution

#### 工作原理
构造器中的计数值（count）实际上就是闭锁需要等待的线程数量。这个值只能被设置一次，而且 CountDownLatch 没有提供任何机制去重新设置这个计数值。

与 CountDownLatch 的第一次交互是主线程等待其他线程。主线程必须在启动其他线程后立即调用 CountDownLatch.await() 方法。
这样主线程的操作就会在这个方法上阻塞，直到其他线程完成各自的任务。

其他N 个线程必须引用闭锁对象，因为他们需要通知 CountDownLatch 对象，他们已经完成了各自的任务。
这种通知机制是通过 CountDownLatch.countDown() 方法来完成的；每调用一次这个方法，在构造函数中初始化的 count 值就减1。
所以当N个线程都调用了这个方法，count 的值等于0，然后主线程就能通过 await() 方法，恢复执行自己的任务。

#### 例子
```
import java.util.concurrent.CountDownLatch;

public class Player implements Runnable {

    private int id;
    private CountDownLatch begin;
    private CountDownLatch end;

    public Player(int i, CountDownLatch begin, CountDownLatch end) {
        super();
        this.id = i;
        this.begin = begin;
        this.end = end;
    }

    @Override
    public void run() {
        try {
            begin.await();        //等待begin的状态为0
            Thread.sleep((long) (Math.random() * 100));    //随机分配时间，即运动员完成时间
            System.out.println("Play" + id + " arrived.");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            end.countDown();    //使end状态减1，最终减至0
        }
    }
}
```

```
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CountDownLatchDemo {

    private static final int PLAYER_AMOUNT = 5;

    public CountDownLatchDemo() {
    }

    /**
     * @param args
     */
    public static void main(String[] args) {
        //对于每位运动员，CountDownLatch减1后即结束比赛
        CountDownLatch begin = new CountDownLatch(1);
        //对于整个比赛，所有运动员结束后才算结束
        CountDownLatch end = new CountDownLatch(PLAYER_AMOUNT);
        Player[] plays = new Player[PLAYER_AMOUNT];

        for (int i = 0; i < PLAYER_AMOUNT; i++){
            plays[i] = new Player(i + 1, begin, end);
        }

        //设置特定的线程池，大小为5
        ExecutorService exe = Executors.newFixedThreadPool(PLAYER_AMOUNT);
        for (final Player p : plays){
            exe.execute(p);            //分配线程
        }
        System.out.println("Race begins!");
        begin.countDown();
        try {
            end.await();            //等待end状态变为0，即为比赛结束
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            System.out.println("Race ends!");
        }
        //exe.shutdown();
    }
}
```

运行结果：
```
Race begins!
Play2 arrived.
Play4 arrived.
Play5 arrived.
Play3 arrived.
Play1 arrived.
Race ends!
```

#### 参考
[什么时候使用CountDownLatch](http://www.importnew.com/15731.html)
