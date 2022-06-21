---
id: 23
title: 'Java Concurrency Tutorial &#8211; Atomicity and race conditions'
date: 2014-08-11T10:00:00+01:00
author: xpadro
layout: post
permalink: /2014/08/java-concurrency-tutorial-atomicity-and-race-conditions.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2014/08/java-concurrency-tutorial-atomicity-and.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/3677990356220144765
categories:
  - Concurrency
  - Java
tags:
  - Concurrency
---

Atomicity and race conditions is one of the key concepts in multi-threaded programs. We say a set of actions is atomic if they all execute as a single operation, in an indivisible manner. Taking for granted that a set of actions in a multi-threaded program will be executed serially may lead to incorrect results. The reason is due to thread interference, which means that if two threads execute several steps on the same data, they may overlap.

&nbsp;

## <span style="color: #003366;">1 Interleaving</span>

The following <a href="https://github.com/xpadro/concurrency/blob/master/basic.atomicity/src/main/java/atomicity/Interleaving.java" target="_blank" rel="noopener">Interleaving</a> example shows two threads executing several actions (prints in a loop) and how they are overlapped:

<pre class="lang:java decode:true ">public class Interleaving {
    
    public void show() {
        for (int i = 0; i &lt; 5; i++) {
            System.out.println(Thread.currentThread().getName() + " - Number: " + i);
        }
    }
    
    public static void main(String[] args) {
        final Interleaving main = new Interleaving();
        
        Runnable runner = new Runnable() {
            @Override
            public void run() {
                main.show();
            }
        };
        
        new Thread(runner, "Thread 1").start();
        new Thread(runner, "Thread 2").start();
    }
}</pre>

&nbsp;

When executed, it will produce unpredictable results. As an example:

<span style="font-size: x-small;">Thread 2 &#8211; Number: 0</span>  
<span style="font-size: x-small;">Thread 2 &#8211; Number: 1</span>  
<span style="font-size: x-small;">Thread 2 &#8211; Number: 2</span>  
<span style="font-size: x-small;">Thread 1 &#8211; Number: 0</span>  
<span style="font-size: x-small;">Thread 1 &#8211; Number: 1</span>  
<span style="font-size: x-small;">Thread 1 &#8211; Number: 2</span>  
<span style="font-size: x-small;">Thread 1 &#8211; Number: 3</span>  
<span style="font-size: x-small;">Thread 1 &#8211; Number: 4</span>  
<span style="font-size: x-small;">Thread 2 &#8211; Number: 3</span>  
<span style="font-size: x-small;">Thread 2 &#8211; Number: 4</span>

In this case, nothing wrong happens since they are just printing numbers. However, when you need to share the state of an object (its data) without synchronization, this leads to the presence of race conditions.

## <span style="color: #0b5394;">2 Race condition</span>

Your code will have a race condition if there’s a possibility to produce incorrect results due to thread interleaving. This section describes two types of race conditions:

  1. Check-then-act
  2. Read-modify-write

To remove race conditions and enforce thread safety, we must make these actions atomic by using synchronization. Examples in the following sections will show what the effects of these race conditions are.

#### <span style="color: #003366;">Check-then-act race condition</span>

This race condition appears when you have a shared field and expect to serially execute the following steps:

  1. Get a value from a field.
  2. Do something based on the result of the previous check.

The problem here is that when the first thread is going to act after the previous check, another thread may have interleaved and changed the value of the field. Now, the first thread will act based on a value that is no longer valid. This is easier seen with an example.

<a href="https://github.com/xpadro/concurrency/blob/master/basic.atomicity/src/main/java/atomicity/UnsafeCheckThenAct.java" target="_blank" rel="noopener">UnsafeCheckThenAct</a> is expected to change the field _number_ once. Following calls to _changeNumber_ method, should result in the execution of the else condition:

<pre class="lang:java decode:true ">public class UnsafeCheckThenAct {
    private int number;
    
    public void changeNumber() {
        if (number == 0) {
            System.out.println(Thread.currentThread().getName() + " | Changed");
            number = -1;
        }
        else {
            System.out.println(Thread.currentThread().getName() + " | Not changed");
        }
    }
    
    public static void main(String[] args) {
        final UnsafeCheckThenAct checkAct = new UnsafeCheckThenAct();
        
        for (int i = 0; i &lt; 50; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    checkAct.changeNumber();
                }
            }, "T" + i).start();
        }
    }
}</pre>

&nbsp;

But since this code is not synchronized, it may (there&#8217;s no guarantee) result in several modifications of the field:

<span style="font-size: x-small;">T13 | Changed</span>  
<span style="font-size: x-small;">T17 | Changed</span>  
<span style="font-size: x-small;">T35 | Not changed</span>  
<span style="font-size: x-small;">T10 | Changed</span>  
<span style="font-size: x-small;">T48 | Not changed</span>  
<span style="font-size: x-small;">T14 | Changed</span>  
<span style="font-size: x-small;">T60 | Not changed</span>  
<span style="font-size: x-small;">T6 | Changed</span>  
<span style="font-size: x-small;">T5 | Changed</span>  
<span style="font-size: x-small;">T63 | Not changed</span>  
<span style="font-size: x-small;">T18 | Not changed</span>

Another example of this race condition is <a href="http://stackoverflow.com/questions/6782660/incorrect-lazy-initialization" target="_blank" rel="noopener">lazy initialization</a>.

&nbsp;

#### <span style="color: #003366;">Check-then-act race condition solution</span>

A simple way to correct this is to use synchronization.

<a href="https://github.com/xpadro/concurrency/blob/master/basic.atomicity/src/main/java/atomicity/SafeCheckThenAct.java" target="_blank" rel="noopener">SafeCheckThenAct</a> is thread-safe because it has removed the race condition by synchronizing all accesses to the shared field:

<pre class="lang:java decode:true ">public class SafeCheckThenAct {
    private int number;
    
    public synchronized void changeNumber() {
        if (number == 0) {
            System.out.println(Thread.currentThread().getName() + " | Changed");
            number = -1;
        }
        else {
            System.out.println(Thread.currentThread().getName() + " | Not changed");
        }
    }
    
    public static void main(String[] args) {
        final SafeCheckThenAct checkAct = new SafeCheckThenAct();
        
        for (int i = 0; i &lt; 50; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    checkAct.changeNumber();
                }
            }, "T" + i).start();
        }
    }
}</pre>

&nbsp;

Now, executing this code will always produce the same expected result; only a single thread will change the field:

<span style="font-size: x-small;">T0 | Changed</span>  
<span style="font-size: x-small;">T54 | Not changed</span>  
<span style="font-size: x-small;">T53 | Not changed</span>  
<span style="font-size: x-small;">T62 | Not changed</span>  
<span style="font-size: x-small;">T52 | Not changed</span>  
<span style="font-size: x-small;">T51 | Not changed</span>  
<span style="font-size: x-small;">&#8230;</span>

In some cases, there will be other mechanisms which perform better than synchronizing the whole method but I won’t discuss them in this post.

#### <span style="color: #003366;">Read-modify-write race condition</span>

Here we have another type of race condition which appears when executing the following set of actions:

  1. Fetch a value from a field.
  2. Modify the value.
  3. Store the new value to the field.

In this case, there’s another dangerous possibility which consists in the loss of some updates to the field. One possible outcome is:

<span style="font-size: x-small;">Field’s value is 1.</span>  
<span style="font-size: x-small;">Thread 1 gets the value from the field (1).</span>  
<span style="font-size: x-small;">Thread 1 modifies the value (5).</span>  
<span style="font-size: x-small;">Thread 2 reads the value from the field (1).</span>  
<span style="font-size: x-small;">Thread 2 modifies the value (7).</span>  
<span style="font-size: x-small;">Thread 1 stores the value to the field (5).</span>  
<span style="font-size: x-small;">Thread 2 stores the value to the field (7).</span>

As you can see, update with the value 5 has been lost.

Let’s see a code sample. <a href="https://github.com/xpadro/concurrency/blob/master/basic.atomicity/src/main/java/atomicity/UnsafeReadModifyWrite.java" target="_blank" rel="noopener">UnsafeReadModifyWrite</a> shares a numeric field which is incremented each time:

<pre class="lang:java decode:true ">public class UnsafeReadModifyWrite {
    private int number;
    
    public void incrementNumber() {
        number++;
    }
    
    public int getNumber() {
        return this.number;
    }
    
    public static void main(String[] args) throws InterruptedException {
        final UnsafeReadModifyWrite rmw = new UnsafeReadModifyWrite();
        
        for (int i = 0; i &lt; 1_000; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    rmw.incrementNumber();
                }
            }, "T" + i).start();
        }
        
        Thread.sleep(6000);
        System.out.println("Final number (should be 1_000): " + rmw.getNumber());
    }
}</pre>

&nbsp;

Can you spot the compound action which causes the race condition?

I’m sure you did, but for completeness, I will explain it anyway. The problem is in the increment (_number++_). This may appear to be a single action but in fact, it is a sequence of three actions (get-increment-write).

When executing this code, we may see that we have lost some updates:

<span style="font-size: x-small;">2014-08-08 09:59:18,859|UnsafeReadModifyWrite|Final number (should be 10_000): 9996</span>

Depending on your computer it will be very difficult to reproduce this update loss, since there’s no guarantee on how threads will interleave. If you can’t reproduce the above example, try <a href="https://github.com/xpadro/concurrency/blob/master/basic.atomicity/src/main/java/atomicity/UnsafeReadModifyWriteWithLatch.java" target="_blank" rel="noopener">UnsafeReadModifyWriteWithLatch</a>, which uses a <a href="http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/CountDownLatch.html" target="_blank" rel="noopener">CountDownLatch</a> to synchronize thread’s start, and repeats the test a hundred times. You should probably see some invalid values among all the results:

<span style="font-size: x-small;">Final number (should be 1_000): 1000</span>  
<span style="font-size: x-small;">Final number (should be 1_000): 1000</span>  
<span style="font-size: x-small;">Final number (should be 1_000): 1000</span>  
<span style="font-size: x-small;">Final number (should be 1_000): 997</span>  
<span style="font-size: x-small;">Final number (should be 1_000): 999</span>  
<span style="font-size: x-small;">Final number (should be 1_000): 1000</span>  
<span style="font-size: x-small;">Final number (should be 1_000): 1000</span>  
<span style="font-size: x-small;">Final number (should be 1_000): 1000</span>  
<span style="font-size: x-small;">Final number (should be 1_000): 1000</span>  
<span style="font-size: x-small;">Final number (should be 1_000): 1000</span>  
<span style="font-size: x-small;">Final number (should be 1_000): 1000</span>

This example can be solved by making all three actions atomic.

&nbsp;

#### <span style="color: #003366;">Read-modify-write race condition solution</span>

<a href="https://github.com/xpadro/concurrency/blob/master/basic.atomicity/src/main/java/atomicity/SafeReadModifyWriteSynchronized.java" target="_blank" rel="noopener">SafeReadModifyWriteSynchronized</a> uses synchronization in all accesses to the shared field:

<pre class="lang:java decode:true ">public class SafeReadModifyWriteSynchronized {
    private int number;
    
    public synchronized void incrementNumber() {
        number++;
    }
    
    public synchronized int getNumber() {
        return this.number;
    }
    
    public static void main(String[] args) throws InterruptedException {
        final SafeReadModifyWriteSynchronized rmw = new SafeReadModifyWriteSynchronized();
        
        for (int i = 0; i &lt; 1_000; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    rmw.incrementNumber();
                }
            }, "T" + i).start();
        }
        
        Thread.sleep(4000);
        System.out.println("Final number (should be 1_000): " + rmw.getNumber());
    }
}</pre>

&nbsp;

Let’s see another example to remove this race condition. In this specific case, and since the field number is independent to other variables, we can make use of atomic variables. <a href="https://github.com/xpadro/concurrency/blob/master/basic.atomicity/src/main/java/atomicity/SafeReadModifyWriteAtomic.java" target="_blank" rel="noopener">SafeReadModifyWriteAtomic</a> uses atomic variables to store the value of the field:

<pre class="lang:java decode:true ">public class SafeReadModifyWriteAtomic {
    private final AtomicInteger number = new AtomicInteger();
    
    public void incrementNumber() {
        number.getAndIncrement();
    }
    
    public int getNumber() {
        return this.number.get();
    }
    
    public static void main(String[] args) throws InterruptedException {
        final SafeReadModifyWriteAtomic rmw = new SafeReadModifyWriteAtomic();
        
        for (int i = 0; i &lt; 1_000; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    rmw.incrementNumber();
                }
            }, "T" + i).start();
        }
        
        Thread.sleep(4000);
        System.out.println("Final number (should be 1_000): " + rmw.getNumber());
    }
}</pre>

&nbsp;

Following posts will further explain mechanisms like locking or atomic variables.

## <span style="color: #0b5394;">3 Conclusion</span>

This post explained some of the risks implied when executing compound actions in non-synchronized multi-threaded programs. To enforce atomicity and prevent thread interleaving, one must use some type of synchronization.

This post is part of the Java Concurrency Tutorial series. Check <a href="http://xpadro.com/2014/09/java-concurrency-tutorial.html" target="_blank" rel="noopener">here</a> to read the rest of the tutorial.

You can take a look at the source code at <a href="https://github.com/xpadro/concurrency/tree/master/basic.atomicity" target="_blank" rel="noopener">github</a>.

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.