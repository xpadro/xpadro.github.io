---
id: 19
title: 'Java Concurrency Tutorial &#8211; Locking: Intrinsic locks'
date: 2014-09-01T15:08:00+01:00
author: xpadro
layout: post
permalink: /2014/09/java-concurrency-tutorial-locking-intrinsic-locks.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2014/09/java-concurrency-tutorial-locking.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/7661370367374633991
categories:
  - Concurrency
  - Java
tags:
  - Concurrency
---

In previous posts we reviewed some of the main risks of sharing data between different threads (like <a href="http://xpadro.com/2014/08/java-concurrency-tutorial-atomicity-and.html" target="_blank" rel="noopener">atomicity</a> and <a href="http://xpadro.com/2014/08/java-concurrency-tutorial-visibility.html" target="_blank" rel="noopener">visibility</a>). Additionally, we learnt how to design classes in order to be shared safely (<a href="http://xpadro.com/2014/08/java-concurrency-tutorial-thread-safe.html" target="_blank" rel="noopener">thread safe designs</a>). In many situations though, we will need to share mutable data, where some threads will write and others will act as readers. It may be the case that you only have one field, independent to others. And this field needs to be shared between different threads. In this case, you may go with atomic variables. For more complex situations you will need synchronisation.

## <span style="color: #003366;">1 The coffee store example</span>

Let’s start with a simple example like a <a href="https://github.com/xpadro/concurrency/blob/master/synchronization.intrinsic/src/main/java/intrinsic/CoffeeStore.java" target="_blank" rel="noopener">CoffeeStore</a>. This class implements a store where clients can buy coffee. When a client buys coffee, a counter is increased in order to keep track of the number of units sold. The store also registers who was the last client to come to the store.

<pre class="lang:java decode:true ">public class CoffeeStore {
    private String lastClient;
    private int soldCoffees;
    
    private void someLongRunningProcess() throws InterruptedException {
        Thread.sleep(3000);
    }
    
    public void buyCoffee(String client) throws InterruptedException {
        someLongRunningProcess();
        
        lastClient = client;
        soldCoffees++;
        System.out.println(client + " bought some coffee");
    }
    
    public int countSoldCoffees() {return soldCoffees;}
    
    public String getLastClient() {return lastClient;}
}</pre>

&nbsp;

#### <span style="color: #003366;">Program execution</span>

In the following program, four clients decide to come to the store to get their coffee:

<pre class="lang:java decode:true ">public static void main(String[] args) throws InterruptedException {
    CoffeeStore store = new CoffeeStore();
    Thread t1 = new Thread(new Client(store, "Mike"));
    Thread t2 = new Thread(new Client(store, "John"));
    Thread t3 = new Thread(new Client(store, "Anna"));
    Thread t4 = new Thread(new Client(store, "Steve"));
    
    long startTime = System.currentTimeMillis();
    t1.start();
    t2.start();
    t3.start();
    t4.start();
    
    t1.join();
    t2.join();
    t3.join();
    t4.join();
    
    long totalTime = System.currentTimeMillis() - startTime;
    System.out.println("Sold coffee: " + store.countSoldCoffees());
    System.out.println("Last client: " + store.getLastClient());
    System.out.println("Total time: " + totalTime + " ms");
}

private static class Client implements Runnable {
    private final String name;
    private final CoffeeStore store;
    
    public Client(CoffeeStore store, String name) {
        this.store = store;
        this.name = name;
    }
    
    @Override
    public void run() {
        try {
            store.buyCoffee(name);
        } catch (InterruptedException e) {
            System.out.println("interrupted sale");
        }
    }
}</pre>

&nbsp;

The main thread will wait for all four client threads to finish, using Thread.join(). Once the clients have left, we should obviously count four coffees sold in our store, but you may get unexpected results like the one above:

<span style="font-size: x-small;">Mike bought some coffee</span>  
<span style="font-size: x-small;">Steve bought some coffee</span>  
<span style="font-size: x-small;">Anna bought some coffee</span>  
<span style="font-size: x-small;">John bought some coffee</span>  
<span style="font-size: x-small;">Sold coffee: 3</span>  
<span style="font-size: x-small;">Last client: Anna</span>  
<span style="font-size: x-small;">Total time: 3001 ms</span>

We lost one unit of coffee, and also the last client (John) is not the one displayed (Anna). The reason is that since our code is not synchronized, threads interleaved. Our _buyCoffee_ operation should be made atomic.

## <span style="color: #003366;">2 How synchronization works</span>

A synchronized block is an area of code which is guarded by a lock. When a thread enters a synchronized block, it needs to acquire its lock. Once acquired, it won’t release it until exiting the block or throwing an exception. In this way, when another thread tries to enter the synchronized block, it won’t be able to acquire its lock until the owner thread releases it.

This is the Java mechanism to ensure that only on thread at a given time is executing a synchronized block of code, ensuring the atomicity of all actions within that block.

Ok, so you use a lock to guard a synchronized block, but what is a lock? The answer is that any Java object can be used as a lock, which is called intrinsic lock. We will now see some examples of these locks when using synchronization.

## <span style="color: #003366;">3 Synchronized methods</span>

Synchronized methods are guarded by two types of locks:

  * **Synchronized instance methods**: The implicit lock is ‘this’, which is the object used to invoke the method. Each instance of this class will use their own lock.

  * **Synchronized static methods**: The lock is the Class object. All instances of this class will use the same lock.

As usual, this is better seen with some code.

First, we are going to synchronize an instance method. This works as follows: We have one instance of the class shared by two threads (Thread-1 and Thread-2), and another instance used by a third thread (Thread-3):

<pre class="lang:java decode:true">public class InstanceMethodExample {
    private static long startTime;
    
    public void start() throws InterruptedException {
        doSomeTask();
    }
    
    public synchronized void doSomeTask() throws InterruptedException {
        long currentTime = System.currentTimeMillis() - startTime;
        System.out.println(Thread.currentThread().getName() + " | Entering method. Current Time: " + currentTime + " ms");
        Thread.sleep(3000);
        System.out.println(Thread.currentThread().getName() + " | Exiting method");
    }
    
    public static void main(String[] args) {
        InstanceMethodExample instance1 = new InstanceMethodExample();
        
        Thread t1 = new Thread(new Worker(instance1), "Thread-1");
        Thread t2 = new Thread(new Worker(instance1), "Thread-2");
        Thread t3 = new Thread(new Worker(new InstanceMethodExample()), "Thread-3");
        
        startTime = System.currentTimeMillis();
        t1.start();
        t2.start();
        t3.start();
    }
    
    private static class Worker implements Runnable {
        private final InstanceMethodExample instance;
        
        public Worker(InstanceMethodExample instance) {
            this.instance = instance;
        }
        
        @Override
        public void run() {
            try {
                instance.start();
            } catch (InterruptedException e) {
                System.out.println(Thread.currentThread().getName() + " interrupted");
            }
        }
    }
}</pre>

&nbsp;

#### <span style="color: #003366;">Program execution</span>

Since _doSomeTask_ method is synchronized, you would expect that only one thread will execute its code at a given time. But that’s wrong, since it is an instance method; different instances will use a different lock as the output demonstrates:

<span style="font-size: x-small;">Thread-1 | Entering method. Current Time: 0 ms</span>  
<span style="font-size: x-small;">Thread-3 | Entering method. Current Time: 1 ms</span>  
<span style="font-size: x-small;">Thread-3 | Exiting method</span>  
<span style="font-size: x-small;">Thread-1 | Exiting method</span>  
<span style="font-size: x-small;">Thread-2 | Entering method. Current Time: 3001 ms</span>  
<span style="font-size: x-small;">Thread-2 | Exiting method</span>

Since Thread-1 and Thread-3 use a different instance (and hence, a different lock), they both enter the block at the same time. On the other hand, Thread-2 uses the same instance (and lock) as Thread-1. Therefore, it has to wait until Thread-1 releases the lock.

Now let’s change the method signature and use a static method. <a href="https://github.com/xpadro/concurrency/blob/master/synchronization.intrinsic/src/main/java/intrinsic/StaticMethodExample.java" target="_blank" rel="noopener">StaticMethodExample</a> has the same code except the following line:

<pre class="lang:java decode:true ">public static synchronized void doSomeTask() throws InterruptedException {</pre>

&nbsp;

If we execute the main method we will get the following output:

<span style="font-size: x-small;">Thread-1 | Entering method. Current Time: 0 ms</span>  
<span style="font-size: x-small;">Thread-1 | Exiting method</span>  
<span style="font-size: x-small;">Thread-3 | Entering method. Current Time: 3001 ms</span>  
<span style="font-size: x-small;">Thread-3 | Exiting method</span>  
<span style="font-size: x-small;">Thread-2 | Entering method. Current Time: 6001 ms</span>  
<span style="font-size: x-small;">Thread-2 | Exiting method</span>

Since the synchronized method is static, the Class object lock guards it. Despite using different instances, all threads will need to acquire the same lock. Hence, any thread will have to wait for the previous thread to release the lock.

## <span style="color: #0b5394;">4 Back to the coffee store example</span>

I have now modified the Coffee Store example in order to synchronize its methods. The result is as follows:

<pre class="lang:java decode:true ">public class SynchronizedCoffeeStore {
    private String lastClient;
    private int soldCoffees;
    
    private void someLongRunningProcess() throws InterruptedException {
        Thread.sleep(3000);
    }
    
    public synchronized void buyCoffee(String client) throws InterruptedException {
        someLongRunningProcess();
        
        lastClient = client;
        soldCoffees++;
        System.out.println(client + " bought some coffee");
    }
    
    public synchronized int countSoldCoffees() {return soldCoffees;}
    
    public synchronized String getLastClient() {return lastClient;}
}</pre>

&nbsp;

Now, if we execute the program, we won’t lose any sale:

<span style="font-size: x-small;">Mike bought some coffee</span>  
<span style="font-size: x-small;">Steve bought some coffee</span>  
<span style="font-size: x-small;">Anna bought some coffee</span>  
<span style="font-size: x-small;">John bought some coffee</span>  
<span style="font-size: x-small;">Sold coffee: 4</span>  
<span style="font-size: x-small;">Last client: John</span>  
<span style="font-size: x-small;">Total time: 12005 ms</span>

Perfect! Well, it really is? Now the program’s execution time is 12 seconds.  You sure have noticed a _someLongRunningProcess_ method executing during each sale. It can be an operation which has nothing to do with the sale, but since we synchronized the whole method, now each thread has to wait for it to execute. Could we leave this code out of the synchronized block? Sure! Have a look at synchronized blocks in the next section.

## <span style="color: #0b5394;">5 Synchronized blocks</span>

The previous section showed us that we may not always need to synchronize the whole method. Since all the synchronized code forces a serialization of all thread executions, we should minimize the length of the synchronized block. In our Coffee store example, we could leave the long running process out of it. In this section’s example, we are going to use synchronized blocks:

In <a href="https://github.com/xpadro/concurrency/blob/master/synchronization.intrinsic/src/main/java/intrinsic/SynchronizedBlockCoffeeStore.java" target="_blank" rel="noopener">SynchronizedBlockCoffeeStore</a>, we modify the _buyCoffee_ method to exclude the long running process outside of the synchronized block:

<pre class="lang:java decode:true ">public void buyCoffee(String client) throws InterruptedException {
    someLongRunningProcess();
    
    synchronized(this) {
        lastClient = client;
        soldCoffees++;
        System.out.println(client + " bought some coffee");
    }
}

public synchronized int countSoldCoffees() {return soldCoffees;}

public synchronized String getLastClient() {return lastClient;}</pre>

&nbsp;

In the previous synchronized block, we use ‘this’ as its lock. It’s the same lock as in synchronized instance methods. Beware of using another lock, since we are using this lock in other methods of this class (_countSoldCoffees_ and _getLastClient_).

Let’s see the result of executing the modified program:

<span style="font-size: x-small;">Mike bought some coffee</span>  
<span style="font-size: x-small;">John bought some coffee</span>  
<span style="font-size: x-small;">Anna bought some coffee</span>  
<span style="font-size: x-small;">Steve bought some coffee</span>  
<span style="font-size: x-small;">Sold coffee: 4</span>  
<span style="font-size: x-small;">Last client: Steve</span>  
<span style="font-size: x-small;">Total time: 3015 ms</span>

We have significantly reduced the duration of the program while keeping the code synchronized.

## <span style="color: #003366;">6 Using private locks</span>

The previous section used a lock on the instance object, but you can use any object as its lock. In this section we are going to use a private lock and see what the risk is of using it.

&nbsp;

#### <span style="color: #003366;">private lock example</span>

In <a href="https://github.com/xpadro/concurrency/blob/master/synchronization.intrinsic/src/main/java/intrinsic/PrivateLockExample.java" target="_blank" rel="noopener">PrivateLockExample</a>, we have a synchronized block guarded by a private lock (myLock):

<pre class="lang:java decode:true ">public class PrivateLockExample {
    private Object myLock = new Object();
    
    public void executeTask() throws InterruptedException {
        synchronized(myLock) {
            System.out.println("executeTask - Entering...");
            Thread.sleep(3000);
            System.out.println("executeTask - Exiting...");
        }
    }
}</pre>

&nbsp;

If one thread enters _executeTask_ method will acquire _myLock_ lock. Any other thread entering other methods within this class guarded by the same _myLock_ lock, will have to wait in order to acquire it.

&nbsp;

#### <span style="color: #003366;">different private locks</span>

But now, let’s imagine that someone wants to extend this class in order to add its own methods, and these methods also need to be synchronized because they need to use the same shared data. Since the lock is private in the base class, the extended class won’t have access to it. If the extended class synchronizes its methods, they will be guarded by ‘this’. In other words, it will use another lock.

<a href="https://github.com/xpadro/concurrency/blob/master/synchronization.intrinsic/src/main/java/intrinsic/MyPrivateLockExample.java" target="_blank" rel="noopener">MyPrivateLockExample</a> extends the previous class and adds its own synchronized method _executeAnotherTask_:

<pre class="lang:java decode:true ">public class MyPrivateLockExample extends PrivateLockExample {
    public synchronized void executeAnotherTask() throws InterruptedException {
        System.out.println("executeAnotherTask - Entering...");
        Thread.sleep(3000);
        System.out.println("executeAnotherTask - Exiting...");
    }
    
    public static void main(String[] args) {
        MyPrivateLockExample privateLock = new MyPrivateLockExample();
        
        Thread t1 = new Thread(new Worker1(privateLock));
        Thread t2 = new Thread(new Worker2(privateLock));
        
        t1.start();
        t2.start();
    }
    
    private static class Worker1 implements Runnable {
        private final MyPrivateLockExample privateLock;
        
        public Worker1(MyPrivateLockExample privateLock) {
            this.privateLock = privateLock;
        }
        
        @Override
        public void run() {
            try {
                privateLock.executeTask();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    
    private static class Worker2 implements Runnable {
        private final MyPrivateLockExample privateLock;
        
        public Worker2(MyPrivateLockExample privateLock) {
            this.privateLock = privateLock;
        }
        
        @Override
        public void run() {
            try {
                privateLock.executeAnotherTask();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}</pre>

&nbsp;

#### <span style="color: #003366;">Program execution</span>

The program uses two worker threads that will execute _executeTask_ and _executeAnotherTask_ respectively. The output shows how threads are interleaved since they are not using the same lock:

<span style="font-size: x-small;">executeTask &#8211; Entering&#8230;</span>  
<span style="font-size: x-small;">executeAnotherTask &#8211; Entering&#8230;</span>  
<span style="font-size: x-small;">executeAnotherTask &#8211; Exiting&#8230;</span>  
<span style="font-size: x-small;">executeTask &#8211; Exiting&#8230;</span>

## <span style="color: #0b5394;">7 Conclusion</span>

We have reviewed the use of intrinsic locks by using Java’s built-in locking mechanism. The main concern here is that synchronized blocks that need to use shared data; have to use the same lock.

This post is part of the Java Concurrency Tutorial series. Check <a href="http://xpadro.com/2014/09/java-concurrency-tutorial.html" target="_blank" rel="noopener">here</a> to read the rest of the tutorial.

You can find the source code at <a href="https://github.com/xpadro/concurrency/tree/master/synchronization.intrinsic" target="_blank" rel="noopener">Github</a>.

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.