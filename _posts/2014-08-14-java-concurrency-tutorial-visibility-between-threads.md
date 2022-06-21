---
id: 22
title: 'Java Concurrency Tutorial &#8211; Visibility between threads'
date: 2014-08-14T10:07:00+01:00
author: xpadro
layout: post
permalink: /2014/08/java-concurrency-tutorial-visibility-between-threads.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2014/08/java-concurrency-tutorial-visibility.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/1976979485810702038
image: /wp-content/uploads/2014/08/noVisibility.png
categories:
  - Concurrency
  - Java
tags:
  - Concurrency
---

When sharing an object’s state between different threads, other issues besides <a href="http://xpadro.com/2014/08/java-concurrency-tutorial-atomicity-and.html" target="_blank" rel="noopener">atomicity</a> come into play. One of them is visibility between threads.

The key fact is that without synchronization, instructions are not guaranteed to be executed in the order in which they appear in your source code. This won’t affect the result in a single-threaded program but, in a multi-threaded program, it is possible that if one thread updates a value, another thread doesn’t see the update when it needs it or doesn’t see it at all.

In a multi-threaded environment, it is the program’s responsibility to identify when data is shared between different threads and act in consequence (using synchronization).

&nbsp;

## <span style="color: #003366;">1 No visibility between threads problem</span>

The example in <a href="https://github.com/xpadro/concurrency/blob/master/basic.visibility/src/main/java/visibility/NoVisibility.java" target="_blank" rel="noopener">NoVisibility</a> consists in two threads that share a flag. The writer thread updates the flag and the reader thread waits until the flag is set:

<pre class="lang:java decode:true ">public class NoVisibility {
    private static boolean ready;
    
    public static void main(String[] args) throws InterruptedException {
        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    if (ready) {
                        System.out.println("Reader Thread - Flag change received. Finishing thread.");
                        break;
                    }
                }
            }
        }).start();
        
        Thread.sleep(3000);
        System.out.println("Writer thread - Changing flag...");
        ready = true;
    }
}</pre>

&nbsp;

This program might result in an infinite loop, since the reader thread may not see the updated flag and wait forever.

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <img loading="lazy" class="alignnone wp-image-180 size-full" src="http://xpadro.com/wp-content/uploads/2014/08/noVisibility-1.png" alt="No visibility between threads console output" width="746" height="78" srcset="https://xpadro.com/wp-content/uploads/2014/08/noVisibility-1.png 746w, https://xpadro.com/wp-content/uploads/2014/08/noVisibility-1-300x31.png 300w" sizes="(max-width: 746px) 100vw, 746px" />
</div>

&nbsp;

## <span style="color: #003366;">2 No visibility between threads solution</span>

With synchronization we can guarantee that this reordering doesn’t take place, avoiding the infinite loop. To ensure visibility we have two options:

  * Locking: Guarantees visibility and atomicity (as long as it uses the same lock).
  * Volatile field: Guarantees visibility.

The volatile keyword acts like some sort of synchronized block. Each time the field is accessed, it will be like entering a synchronized block. The main difference is that it doesn’t use locks. For this reason, it may be suitable for examples like the above one (updating a shared flag) but not when using compound actions.

We will now modify the previous example by adding the volatile keyword to the ready field.

<pre class="lang:java decode:true ">public class Visibility {
    private static volatile boolean ready;
    
    public static void main(String[] args) throws InterruptedException {
        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    if (ready) {
                        System.out.println("Reader Thread - Flag change received. Finishing thread.");
                        break;
                    }
                }
            }
        }).start();
        
        Thread.sleep(3000);
        System.out.println("Writer thread - Changing flag...");
        ready = true;
    }
}</pre>

&nbsp;

<a href="https://github.com/xpadro/concurrency/blob/master/basic.visibility/src/main/java/visibility/Visibility.java" target="_blank" rel="noopener">Visibility</a> will not result in an infinite loop anymore. Updates made by the writer thread will be visible to the reader thread:

<span style="font-size: x-small;">Writer thread &#8211; Changing flag&#8230;</span>  
<span style="font-size: x-small;">Reader Thread &#8211; Flag change received. Finishing thread.</span>

## <span style="color: #0b5394;">3 Conclusion</span>

We learned about another risk when sharing data in multi-threaded programs. For a simple example like the one shown here, we can simply use a volatile field. Other situations will require us to use atomic variables or locking.

This post is part of the Java Concurrency Tutorial series. Check my <a href="http://xpadro.com/2014/09/java-concurrency-tutorial.html" target="_blank" rel="noopener">Java Concurrency Tutorial</a> to read the rest of the tutorial.

You can take a look at the source code at <a href="https://github.com/xpadro/concurrency/tree/master/basic.visibility" target="_blank" rel="noopener">github</a>.

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.

&nbsp;