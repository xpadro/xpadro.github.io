---
id: 21
title: 'Java Concurrency Tutorial &#8211; Thread-safe designs'
date: 2014-08-25T11:47:00+01:00
author: xpadro
layout: post
permalink: /2014/08/java-concurrency-tutorial-thread-safe-designs.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2014/08/java-concurrency-tutorial-thread-safe.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/2434657450135498344
categories:
  - Concurrency
  - Java
tags:
  - Concurrency
---

After reviewing what the main risks are when dealing with concurrent programs (like <a href="http://xpadro.com/2014/08/java-concurrency-tutorial-atomicity-and.html" target="_blank" rel="noopener">atomicity</a> or <a href="http://xpadro.com/2014/08/java-concurrency-tutorial-visibility.html" target="_blank" rel="noopener">visibility</a>), we will go through some class designs that will help us prevent the aforementioned bugs. Some of these designs result in the construction of thread-safe concurrency solutions. This will allow us to share objects safely between threads. As an example, we will consider immutable and stateless objects. Other designs will prevent different threads from modifying the same data, like thread-local variables.

You can see all the source code at <a href="https://github.com/xpadro/concurrency/tree/master/basic.threadsafety" target="_blank" rel="noopener">github</a>.

## <span style="color: #003366;">1 Immutable objects</span>

Immutable objects have a state (have data which represent the object&#8217;s state), but it is built upon construction, and once the object is instantiated, the state cannot be modified.

Although threads may interleave, the object has only one possible state. Since all fields are read-only, not a single thread will be able to change object&#8217;s data. For this reason, an immutable object is inherently thread-safe.

<a href="https://github.com/xpadro/concurrency/blob/master/basic.threadsafety/src/main/java/threadsafety/Product.java" target="_blank" rel="noopener">Product</a> shows an example of an immutable class. It builds all its data during construction and none of its fields are modifiable:

<pre class="lang:java decode:true ">public final class Product {
    private final String id;
    private final String name;
    private final double price;
    
    public Product(String id, String name, double price) {
        this.id = id;
        this.name = name;
        this.price = price;
    }
    
    public String getId() {
        return this.id;
    }
    
    public String getName() {
        return this.name;
    }
    
    public double getPrice() {
        return this.price;
    }
    
    public String toString() {
        return new StringBuilder(this.id).append("-").append(this.name)
            .append(" (").append(this.price).append(")").toString();
    }
    
    public boolean equals(Object x) {
        if (this == x) return true;
        if (x == null) return false;
        if (this.getClass() != x.getClass()) return false;
        Product that = (Product) x;
        if (!this.id.equals(that.id)) return false;
        if (!this.name.equals(that.name)) return false;
        if (this.price != that.price) return false;
        
        return true;
    }
    
    public int hashCode() {
        int hash = 17;
        hash = 31 * hash + this.getId().hashCode();
        hash = 31 * hash + this.getName().hashCode();
        hash = 31 * hash + ((Double) this.getPrice()).hashCode();
        
        return hash;
    }
}</pre>

&nbsp;

In some cases, it won&#8217;t be sufficient to make a field final. For example, <a href="https://github.com/xpadro/concurrency/blob/master/basic.threadsafety/src/main/java/threadsafety/MutableProduct.java" target="_blank" rel="noopener">MutableProduct</a> class is not immutable although all fields are final:

<pre class="lang:java decode:true ">public final class MutableProduct {
    private final String id;
    private final String name;
    private final double price;
    private final List&lt;String&gt; categories = new ArrayList&lt;&gt;();
    
    public MutableProduct(String id, String name, double price) {
        this.id = id;
        this.name = name;
        this.price = price;
        this.categories.add("A");
        this.categories.add("B");
        this.categories.add("C");
    }
    
    public String getId() {
        return this.id;
    }
    
    public String getName() {
        return this.name;
    }
    
    public double getPrice() {
        return this.price;
    }
    
    public List&lt;String&gt; getCategories() {
        return this.categories;
    }
    
    public List&lt;String&gt; getCategoriesUnmodifiable() {
        return Collections.unmodifiableList(categories);
    }
    
    public String toString() {
        return new StringBuilder(this.id).append("-").append(this.name)
            .append(" (").append(this.price).append(")").toString();
    }
}</pre>

&nbsp;

Why is the above class not immutable? The reason is we let a reference to escape from the scope of its class. The field &#8216;_categories_&#8216; is a mutable reference, so after returning it, the client could modify it. In order to show this, consider the following program:

<pre class="lang:java decode:true ">public static void main(String[] args) {
    MutableProduct p = new MutableProduct("1", "a product", 43.00);
    
    System.out.println("Product categories");
    for (String c : p.getCategories()) System.out.println(c);
    
    p.getCategories().remove(0);
    System.out.println("\nModified Product categories");
    for (String c : p.getCategories()) System.out.println(c);
}</pre>

&nbsp;

And the console output:

<span style="font-size: x-small;">Product categories</span>  
<span style="font-size: x-small;">A</span>  
<span style="font-size: x-small;">B</span>  
<span style="font-size: x-small;">C</span>  
<span style="font-size: x-small;"><br /> </span><span style="font-size: x-small;">Modified Product categories</span>  
<span style="font-size: x-small;">B</span>  
<span style="font-size: x-small;">C</span>

Since _categories_ field is mutable and it escaped the object&#8217;s scope, the client has modified the categories list. The product, which was supposed to be immutable, has been modified, leading to a new state.

If you want to expose the content of the list, you could use an unmodifiable view of the list:

<pre class="lang:java decode:true">public List&lt;String&gt; getCategoriesUnmodifiable() {
    return Collections.unmodifiableList(categories);
}</pre>

&nbsp;

## <span style="color: #003366;">2 Stateless objects</span>

Stateless objects are similar to immutable objects but in this case, they do not have a state, not even one. When an object is stateless it does not have to remember any data between invocations.

Since there is no state to modify, one thread will not be able to affect the result of another thread invoking the object&#8217;s operations. For this reason, a stateless class is inherently thread-safe.

<a href="https://github.com/xpadro/concurrency/blob/master/basic.threadsafety/src/main/java/threadsafety/ProductHandler.java" target="_blank" rel="noopener">ProductHandler</a> is an example of this type of objects. It contains several operations over Product objects and it does not store any data between invocations. The result of an operation does not depend on previous invocations or any stored data:

<pre class="lang:java decode:true ">public class ProductHandler {
    private static final int DISCOUNT = 90;
    
    public Product applyDiscount(Product p) {
        double finalPrice = p.getPrice() * DISCOUNT / 100;
        
        return new Product(p.getId(), p.getName(), finalPrice);
    }
    
    public double sumCart(List&lt;Product&gt; cart) {
        double total = 0.0;
        for (Product p : cart.toArray(new Product[0])) total += p.getPrice();
        
        return total;
    }
}</pre>

&nbsp;

In its _sumCart_ method, the _ProductHandler_ converts the product list to an array since for-each loop uses an iterator internally to iterate through its elements. List iterators are not thread-safe and could throw a <a href="http://docs.oracle.com/javase/7/docs/api/java/util/ConcurrentModificationException.html" target="_blank" rel="noopener">ConcurrentModificationException</a> if modified during iteration. Depending on your needs, you might choose a different <a href="http://stackoverflow.com/questions/4517653/thread-safe-iteration-over-a-collection" target="_blank" rel="noopener">strategy</a>.

## <span style="color: #003366;">3 Thread-local variables</span>

Thread-local variables are those variables defined within the scope of a thread. No other threads will see nor modify them.

The first type is local variables. In the below example, the _total_ variable is stored in the thread&#8217;s stack:

<pre class="lang:java decode:true ">public double sumCart(List&lt;Product&gt; cart) {
    double total = 0.0;
    for (Product p : cart.toArray(new Product[0])) total += p.getPrice();
    
    return total;
}</pre>

&nbsp;

Just take into account that if instead of a primitive you define a reference and return it, it will escape its scope. You may not know where the returned reference is stored. The code that calls _sumCart_ method could store it in a static field and allow it being shared between different threads.

The second type is <a href="http://docs.oracle.com/javase/7/docs/api/java/lang/ThreadLocal.html" target="_blank" rel="noopener">ThreadLocal</a> class. This class provides a storage independent for each thread. Values stored into an instance of ThreadLocal are accessible from any code within the same thread.

The <a href="https://github.com/xpadro/concurrency/blob/master/basic.threadsafety/src/main/java/threadsafety/ClientRequestId.java" target="_blank" rel="noopener">ClientRequestId</a> class shows an example of ThreadLocal usage:

<pre class="lang:java decode:true ">public class ClientRequestId {
    private static final ThreadLocal&lt;String&gt; id = new ThreadLocal&lt;String&gt;() {
        @Override
        protected String initialValue() {
            return UUID.randomUUID().toString();
        }
    };
    
    public static String get() {
        return id.get();
    }
}</pre>

&nbsp;

The <a href="https://github.com/xpadro/concurrency/blob/master/basic.threadsafety/src/main/java/threadsafety/ProductHandlerThreadLocal.java" target="_blank" rel="noopener">ProductHandlerThreadLocal</a> class uses ClientRequestId to return the same generated id within the same thread:

<pre class="lang:java decode:true ">public class ProductHandlerThreadLocal {
    //Same methods as in ProductHandler class
    
    public String generateOrderId() {
        return ClientRequestId.get();
    }
}</pre>

&nbsp;

If you execute the main method, the console output will show different ids for each thread. As an example:

<span style="font-size: x-small;">T1 &#8211; 23dccaa2-8f34-43ec-bbfa-01cec5df3258</span>  
<span style="font-size: x-small;">T2 &#8211; 936d0d9d-b507-46c0-a264-4b51ac3f527d</span>  
<span style="font-size: x-small;">T2 &#8211; 936d0d9d-b507-46c0-a264-4b51ac3f527d</span>  
<span style="font-size: x-small;">T3 &#8211; 126b8359-3bcc-46b9-859a-d305aff22c7e</span>  
<span style="font-size: x-small;">&#8230;</span>

If you are going to use ThreadLocal, you should care about some of the risks of using it when threads are pooled (like in application servers). You could end up with memory leaks or information leaking between requests. I won&#8217;t extend myself in this subject since the post <a href="https://plumbr.eu/blog/how-to-shoot-yourself-in-foot-with-threadlocals" target="_blank" rel="noopener">How to shoot yourself in foot with ThreadLocals</a> explains well how this can happen.

## <span style="color: #003366;">4 Using synchronization</span>

Another way of providing thread-safe access to objects is through synchronization. If we synchronize all accesses to a reference, only a single thread will access it at a given time. We will discuss this on further posts.

## <span style="color: #003366;">5 Conclusion</span>

We have seen several techniques that help us build simpler objects that can be shared safely between threads. It is much harder to prevent concurrent bugs if an object can have multiple states. On the other hand, if an object can have only one state or none, we won&#8217;t have to worry about different threads accessing it at the same time.

This post is part of the Java Concurrency Tutorial series. Check <a href="http://xpadro.com/2014/09/java-concurrency-tutorial.html" target="_blank" rel="noopener">here</a> to read the rest of the tutorial.

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.