---
id: 16
title: 'Java Concurrency Tutorial &#8211; Locking: Explicit locks'
date: 2015-02-15T18:58:00+01:00
author: xpadro
layout: post
permalink: /2015/02/java-concurrency-tutorial-locking-explicit-locks.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2015/02/java-concurrency-tutorial-locking.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/747386282712066308
image: /wp-content/uploads/2015/02/explicitLocking.png
categories:
  - Concurrency
  - Java
tags:
  - Concurrency
---

In many cases, using implicit locking is enough. Other times, we will need more complex functionalities. In such cases, _java.util.concurrent.locks_ package provides us with lock objects. When it comes to memory synchronization, the internal mechanism of these locks is the same as with implicit locks. The difference is that explicit locks offer additional features.

The main advantages or improvements over implicit synchronization are:

  * Separation of locks by read or write.
  * Some locks allow concurrent access to a shared resource (<a href="http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReadWriteLock.html" target="_blank" rel="noopener">ReadWriteLock</a>).
  * Different ways of acquiring a lock: 
      * Blocking: lock()
      * Non-blocking: tryLock()
      * Interruptible: lockInterruptibly()

&nbsp;

## <span style="color: #0b5394;">1 Classification of lock objects</span>

Lock objects implement one of the following two interfaces:

  * <a href="http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/Lock.html" target="_blank" rel="noopener">Lock</a>: Defines the basic functionalities that a lock object must implement. Basically, this means acquiring and releasing the lock. In contrast to implicit locks, this one allows the acquisition of a lock in a non-blocking or interruptible way (additionally to the blocking way). Main implementations: 
      * ReentrantLock
      * ReadLock (used by ReentrantReadWriteLock)
      * WriteLock (used by ReentrantReadWriteLock)

&nbsp;

  * <a href="http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReadWriteLock.html" target="_blank" rel="noopener">ReadWriteLock</a>: It keeps a pair of locks, one for read-only operations and another one for writing. The read lock can be acquired simultaneously by different reader threads (as long as the resource isn’t already acquired by a write lock), while the write lock is exclusive. In this way, we can have several threads reading the resource concurrently as long as there is not a writing operation. Main implementations: 
      * ReentrantReadWriteLock

The following class diagram shows the relation among the different lock classes:

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <img loading="lazy" class="alignnone wp-image-196 size-full" src="http://xpadro.com/wp-content/uploads/2015/02/explicitLocking-1.png" alt="explicit locks class diagram" width="505" height="455" srcset="https://xpadro.com/wp-content/uploads/2015/02/explicitLocking-1.png 505w, https://xpadro.com/wp-content/uploads/2015/02/explicitLocking-1-300x270.png 300w" sizes="(max-width: 505px) 100vw, 505px" />
</div>

&nbsp;

## <span style="color: #0b5394;">2 ReentrantLock</span>

This lock works the same way as the synchronized block; one thread acquires the lock as long as it is not already acquired by another thread, and it does not release it until unlock is invoked. If the lock is already acquired by another thread, then the thread trying to acquire it becomes blocked until the other thread releases it.

We are going to start with a simple example without locking, and then we will add a reentrant lock to see how it works.

<pre class="lang:java decode:true ">public class NoLocking {
    public static void main(String[] args) {
        Worker worker = new Worker();
        
        Thread t1 = new Thread(worker, "Thread-1");
        Thread t2 = new Thread(worker, "Thread-2");
        t1.start();
        t2.start();
    }
    
    private static class Worker implements Runnable {
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + " - 1");
            System.out.println(Thread.currentThread().getName() + " - 2");
            System.out.println(Thread.currentThread().getName() + " - 3");
        }
    }
}</pre>

&nbsp;

Since the code above is not synchronized, threads will be interleaved. Let’s see the output:

<span style="font-size: x-small;">Thread-2 &#8211; 1</span>  
<span style="font-size: x-small;">Thread-1 &#8211; 1</span>  
<span style="font-size: x-small;">Thread-1 &#8211; 2</span>  
<span style="font-size: x-small;">Thread-1 &#8211; 3</span>  
<span style="font-size: x-small;">Thread-2 &#8211; 2</span>  
<span style="font-size: x-small;">Thread-2 &#8211; 3</span>

Now, we will add a reentrant lock in order to serialize the access to the run method:

<pre class="lang:java decode:true ">public class ReentrantLockExample {
    public static void main(String[] args) {
        Worker worker = new Worker();
        
        Thread t1 = new Thread(worker, "Thread-1");
        Thread t2 = new Thread(worker, "Thread-2");
        t1.start();
        t2.start();
    }
    
    private static class Worker implements Runnable {
        private final ReentrantLock lock = new ReentrantLock();
        
        @Override
        public void run() {
            lock.lock();
            try {
                System.out.println(Thread.currentThread().getName() + " - 1");
                System.out.println(Thread.currentThread().getName() + " - 2");
                System.out.println(Thread.currentThread().getName() + " - 3");
            } finally {
                lock.unlock();
            }
        }
    }
}</pre>

&nbsp;

The above code will safely be executed without threads being interleaved. You may realize that we could have used a synchronized block and the effect would be the same. The question that arises now is what advantages does the reentrant lock provides us?

The main advantages of using this type of lock are described below:

  * Additional ways of acquiring the lock are provided by implementing Lock interface: 
      * **lockInterruptibly**: The current thread will try to acquire de lock and become blocked if another thread owns the lock, like with the lock() method. However, if another thread interrupts the current thread, the acquisition will be cancelled.
      * **tryLock**: It will try to acquire the lock and return immediately, regardless of the lock status. This will prevent the current thread from being blocked if the lock is already acquired by another thread. You can also set the time the current thread will wait before returning (we will see an example of this).
      * **newCondition**: Allows the thread which owns the lock to wait for a specified condition.

  * Additional methods provided by the <a href="http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReentrantLock.html" target="_blank" rel="noopener">ReentrantLock</a> class, primarily for monitoring or testing. For example, _getHoldCount_ or _isHeldByCurrentThread_ methods.

Let’s look at an example using tryLock before moving on to the next lock class.

### <span style="color: #0b5394;">2.1 Trying lock acquisition</span>

In the following example, we have got two threads, trying to acquire the same two locks.

One thread acquires _lock2_ and then it blocks trying to acquire _lock1_:

<pre class="lang:java decode:true ">public void lockBlocking() {
    LOGGER.info("{}|Trying to acquire lock2...", Thread.currentThread().getName());
    lock2.lock();
    try {
        LOGGER.info("{}|Lock2 acquired. Trying to acquire lock1...", Thread.currentThread().getName());
        lock1.lock();
        LOGGER.info("{}|Both locks acquired", Thread.currentThread().getName());
    } finally {
        lock1.unlock();
        lock2.unlock();
    }
}</pre>

&nbsp;

Another thread, acquires _lock1_ and then it tries to acquire _lock2_.

<pre class="lang:java decode:true ">public void lockWithTry() {
    LOGGER.info("{}|Trying to acquire lock1...", Thread.currentThread().getName());
    lock1.lock();
    try {
        LOGGER.info("{}|Lock1 acquired. Trying to acquire lock2...", Thread.currentThread().getName());
        boolean acquired = lock2.tryLock(4, TimeUnit.SECONDS);
        if (acquired) {
            try {
                LOGGER.info("{}|Both locks acquired", Thread.currentThread().getName());
            } finally {
                lock2.unlock();
            }
        }
        else {
            LOGGER.info("{}|Failed acquiring lock2. Releasing lock1", Thread.currentThread().getName());
        }
    } catch (InterruptedException e) {
        //handle interrupted exception
    } finally {
        lock1.unlock();
    }
}</pre>

&nbsp;

Using the standard lock method, this would cause a dead lock, since each thread would be waiting forever for the other to release the lock. However, this time we are trying to acquire it with _tryLock_ specifying a timeout. If it doesn’t succeed after four seconds, it will cancel the action and release the first lock. This will allow the other thread to unblock and acquire both locks.

Let’s see the [full](https://github.com/xpadro/concurrency/blob/master/synchronization.explicit/src/main/java/xpadro/java/concurrency/TryLock.java) example:

<pre class="lang:java decode:true ">public class TryLock {
    private static final Logger LOGGER = LoggerFactory.getLogger(TryLock.class);
    private final ReentrantLock lock1 = new ReentrantLock();
    private final ReentrantLock lock2 = new ReentrantLock();
    
    public static void main(String[] args) {
        TryLock app = new TryLock();
        Thread t1 = new Thread(new Worker1(app), "Thread-1");
        Thread t2 = new Thread(new Worker2(app), "Thread-2");
        t1.start();
        t2.start();
    }
    
    public void lockWithTry() {
        LOGGER.info("{}|Trying to acquire lock1...", Thread.currentThread().getName());
        lock1.lock();
        try {
            LOGGER.info("{}|Lock1 acquired. Trying to acquire lock2...", Thread.currentThread().getName());
            boolean acquired = lock2.tryLock(4, TimeUnit.SECONDS);
            if (acquired) {
                try {
                    LOGGER.info("{}|Both locks acquired", Thread.currentThread().getName());
                } finally {
                    lock2.unlock();
                }
            }
            else {
                LOGGER.info("{}|Failed acquiring lock2. Releasing lock1", Thread.currentThread().getName());
            }
        } catch (InterruptedException e) {
            //handle interrupted exception
        } finally {
            lock1.unlock();
        }
    }
    
    public void lockBlocking() {
        LOGGER.info("{}|Trying to acquire lock2...", Thread.currentThread().getName());
        lock2.lock();
        try {
            LOGGER.info("{}|Lock2 acquired. Trying to acquire lock1...", Thread.currentThread().getName());
            lock1.lock();
            LOGGER.info("{}|Both locks acquired", Thread.currentThread().getName());
        } finally {
            lock1.unlock();
            lock2.unlock();
        }
    }
    
    private static class Worker1 implements Runnable {
        private final TryLock app;
        
        public Worker1(TryLock app) {
            this.app = app;
        }
        
        @Override
        public void run() {
            app.lockWithTry();
        }
    }
    
    private static class Worker2 implements Runnable {
        private final TryLock app;
        
        public Worker2(TryLock app) {
            this.app = app;
        }
        
        @Override
        public void run() {
            app.lockBlocking();
        }
    }
}</pre>

&nbsp;

If we execute the code it will result in the following output:

<span style="font-size: x-small;">13:06:38,654|Thread-2|Trying to acquire lock2&#8230;</span>  
<span style="font-size: x-small;">13:06:38,654|Thread-1|Trying to acquire lock1&#8230;</span>  
<span style="font-size: x-small;">13:06:38,655|Thread-2|Lock2 acquired. Trying to acquire lock1&#8230;</span>  
<span style="font-size: x-small;">13:06:38,655|Thread-1|Lock1 acquired. Trying to acquire lock2&#8230;</span>  
<span style="font-size: x-small;">13:06:42,658|Thread-1|Failed acquiring lock2. Releasing lock1</span>  
<span style="font-size: x-small;">13:06:42,658|Thread-2|Both locks acquired</span>

After the fourth line, each thread has acquired one lock and is blocked trying to acquire the other lock. At the next line, you can notice the four second lapse. Since we reached the timeout, the first thread fails to acquire the lock and releases the one it had already acquired, allowing the second thread to continue.

## <span style="color: #0b5394;">3 ReentrantReadWriteLock</span>

This type of lock keeps a pair of internal locks (a _ReadLock_ and a _WriteLock_). As explained with the interface, this lock allows several threads to read from the resource concurrently. This is specially convenient when having  a resource that has frequent reads but few writes. As long as there isn’t a thread that needs to write, the resource will be concurrently accessed.

The following example shows three threads concurrently reading from a shared resource. When a fourth thread needs to write, it will exclusively lock the resource, preventing reading threads from accessing it while it is writing. Once the write finishes and the lock is released, all reader threads will continue to access the resource concurrently:

<pre class="lang:java decode:true ">public class ReadWriteLockExample {
    private static final Logger LOGGER = LoggerFactory.getLogger(ReadWriteLockExample.class);
    final ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    private Data data = new Data("default value");
    
    public static void main(String[] args) {
        ReadWriteLockExample example = new ReadWriteLockExample();
        example.start();
    }
    
    private void start() {
        ExecutorService service = Executors.newFixedThreadPool(4);
        for (int i=0; i&lt;3; i++) service.execute(new ReadWorker());
        service.execute(new WriteWorker());
        service.shutdown();
    }
    
    class ReadWorker implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i &lt; 2; i++) {
                readWriteLock.readLock().lock();
                try {
                    LOGGER.info("{}|Read lock acquired", Thread.currentThread().getName());
                    Thread.sleep(3000);
                    LOGGER.info("{}|Reading data: {}", Thread.currentThread().getName(), data.getValue());
                } catch (InterruptedException e) {
                    //handle interrupted
                } finally {
                    readWriteLock.readLock().unlock();
                }
            }
        }
    }
    
    class WriteWorker implements Runnable {
        @Override
        public void run() {
            readWriteLock.writeLock().lock();
            try {
                LOGGER.info("{}|Write lock acquired", Thread.currentThread().getName());
                Thread.sleep(3000);
                data.setValue("changed value");
                LOGGER.info("{}|Writing data: changed value", Thread.currentThread().getName());
            } catch (InterruptedException e) {
                //handle interrupted
            } finally {
                readWriteLock.writeLock().unlock();
            }
        }
    }
}</pre>

&nbsp;

The console output shows the result:

<span style="font-size: x-small;">11:55:01,632|pool-1-thread-1|Read lock acquired</span>  
<span style="font-size: x-small;">11:55:01,632|pool-1-thread-2|Read lock acquired</span>  
<span style="font-size: x-small;">11:55:01,632|pool-1-thread-3|Read lock acquired</span>  
<span style="font-size: x-small;">11:55:04,633|pool-1-thread-3|Reading data: default value</span>  
<span style="font-size: x-small;">11:55:04,633|pool-1-thread-1|Reading data: default value</span>  
<span style="font-size: x-small;">11:55:04,633|pool-1-thread-2|Reading data: default value</span>  
<span style="font-size: x-small;">11:55:04,634|pool-1-thread-4|Write lock acquired</span>  
<span style="font-size: x-small;">11:55:07,634|pool-1-thread-4|Writing data: changed value</span>  
<span style="font-size: x-small;">11:55:07,634|pool-1-thread-3|Read lock acquired</span>  
<span style="font-size: x-small;">11:55:07,635|pool-1-thread-1|Read lock acquired</span>  
<span style="font-size: x-small;">11:55:07,635|pool-1-thread-2|Read lock acquired</span>  
<span style="font-size: x-small;">11:55:10,636|pool-1-thread-3|Reading data: changed value</span>  
<span style="font-size: x-small;">11:55:10,636|pool-1-thread-1|Reading data: changed value</span>  
<span style="font-size: x-small;">11:55:10,636|pool-1-thread-2|Reading data: changed value</span>

As you can see, when writer thread acquires the write lock (thread-4), no other threads can access the resource.

## <span style="color: #0b5394;">4 Conclusion</span>

This post shows which are the main implementations of explicit locks and explains some of its improved features with respect to implicit locking.

This post is part of the Java Concurrency Tutorial series. Check <a href="http://xpadro.com/2014/09/java-concurrency-tutorial.html" target="_blank" rel="noopener">here</a> to read the rest of the tutorial.

You can find the source code at <a href="https://github.com/xpadro/concurrency/tree/master/synchronization.explicit" target="_blank" rel="noopener">Github</a>.

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.