---
id: 20
title: Java Concurrency Tutorial
date: 2014-09-01T15:06:00+01:00
author: xpadro
layout: post
permalink: /2014/09/java-concurrency-tutorial.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2014/09/java-concurrency-tutorial.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/3664554948952418810
categories:
  - Concurrency
  - Java
tags:
  - Concurrency
---
This Java concurrency tutorial consists of several posts that explain the main concepts of concurrency in Java. It starts with the basics with posts about the main concerns or risks of using non synchronised programs.

This tutorial consists on two main areas:

  * Basic concepts about how concurrency works
  * This section explains different synchronisation mechanisms

Therefore, each section concept links to an article explaining it with examples. In addition, each article contains examples based on source code you can check on Github.

&nbsp;

## <span style="color: #0b5394;">Basics</span>

**<a style="color: #0b5394;" href="http://xpadro.com/2014/08/java-concurrency-tutorial-atomicity-and.html" target="_blank" rel="noopener">Atomicity and race conditions</a>**  
Atomicity is one of the main concerns in concurrent programs. This post shows the effects of executing compound actions in non-synchronized code.

**<a style="color: #0b5394;" href="http://xpadro.com/2014/08/java-concurrency-tutorial-visibility.html" target="_blank" rel="noopener">Visibility between threads</a>**  
Another of the risks of executing code concurrently. How values written by one thread can become visible to other threads accessing the same data.

**<a style="color: #0b5394;" href="http://xpadro.com/2014/08/java-concurrency-tutorial-thread-safe.html" target="_blank" rel="noopener">Thread-safe designs</a>**  
After looking at the main risks of sharing data, this post describes several class designs that can be shared safely between different threads.

## <span style="color: #0b5394;">Synchronization</span>

**<a style="color: #0b5394;" href="http://xpadro.com/2014/09/java-concurrency-tutorial-locking.html" target="_blank" rel="noopener">Locking &#8211; Intrinsic locks</a>**  
Intrinsic locks are Java&#8217;s built-in mechanism for locking. It ensures that compound actions within a synchronized block are atomic and create a happens-before relationship.

**<span style="color: #0b5394;"><a href="http://xpadro.com/2015/02/java-concurrency-tutorial-locking.html" target="_blank" rel="noopener">Locking &#8211; Explicit locks</a></span>**  
Explicit locks provide additional features to the Java synchronisation mechanism. Here, we take a look at the main implementations and how they work.

&nbsp;