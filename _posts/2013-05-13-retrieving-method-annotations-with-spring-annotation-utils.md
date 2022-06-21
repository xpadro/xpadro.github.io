---
id: 37
title: Retrieving method annotations with Spring AnnotationUtils
date: 2013-05-13T12:27:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/05/13/retrieving-method-annotations-with-spring-annotation-utils/
permalink: /2013/05/retrieving-method-annotations-with-spring-annotation-utils.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/05/retrieving-method-annotations-with_13.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/3600879544709502624
categories:
  - Spring
tags:
  - Reflection
  - Spring
---

The JDK provides us with several lookup methods that allow us to retrieve annotations from a class, method, field or added to method parameters. The Spring [AnnotationUtils](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/annotation/AnnotationUtils.html) is a general utility class for annotations which extends the basic functionalities. In this post I will explain the main features of this class, focusing on retrieving annotations from a method.

## <span style="color: #073763;">1 getAnnotation method</span>

Signature:

<pre class="lang:java decode:true ">public static &lt;A extends Annotation&gt; A getAnnotation(Method method, Class&lt;A&gt; annotationType)</pre>

This method looks up if _method_ contains the specified annotation type and returns it if found. So, what&#8217;s the difference between the JDK _getAnnotation_ method? The Spring version handles bridge methods. You can check <a href="http://docs.oracle.com/javase/tutorial/java/generics/bridgeMethods.html" target="_blank" rel="noopener"><i>effects of Type Erasure and Bridge Methods</i></a> tutorial to get a good explanation.

Let&#8217;s see it with an example. Imagine we got a generic class _Foo_, and a _Bar_ class that extends it:

<pre class="lang:java decode:true ">public class Foo&lt;T&gt; {
    public T aMethod(T param) {
        return param;
    }
}

public class Bar extends Foo&lt;String&gt; {
    @MyAnnotation
    public String aMethod(String param) {
        return param;
    }
}</pre>

&nbsp;

Now we try to retrieve _@MyAnnotation_ with both utility methods:

<pre class="lang:java decode:true">Method barAMethod = Bar.class.getMethod("aMethod", Object.class);

System.out.println("getAnnotation (JDK): " + barAMethod.getAnnotation(MyAnnotation.class));
System.out.println("getAnnotation (Spring): " + AnnotationUtils.getAnnotation(barAMethod, MyAnnotation.class));</pre>

The result is as follows:

<div>
  <p>
    <a style="margin-left: 1em; margin-right: 1em;" href="http://1.bp.blogspot.com/-t3zcJO-5qZw/VUZApu9P2bI/AAAAAAAAEkI/heX9ap0lk40/s1600/annot1.png"><img loading="lazy" class="alignnone" src="http://1.bp.blogspot.com/-t3zcJO-5qZw/VUZApu9P2bI/AAAAAAAAEkI/heX9ap0lk40/s1600/annot1.png" alt="getAnnotation result" width="492" height="37" border="0" /></a>
  </p>
</div>

## <span style="color: #073763;">2 findAnnotation method</span>

Signature:

<pre class="lang:java decode:true ">public static &lt;A extends Annotation&gt; A findAnnotation(Method method, Class&lt;A&gt; annotationType)</pre>

This method also retrieves the specified annotation type from _method_. The difference between _getAnnotation_ is that in this case, it will look up the entire inheritance hierarchy of the specified method.

Let&#8217;s see another example. We got the classes below:

<pre class="lang:java decode:true ">public class Foo {
    @MyAnnotation
    public void anotherMethod() {
        //...
    }
}

public class Bar extends Foo {
    @Override
    public void anotherMethod() {
        //...
    }
}</pre>

Trying to retrieve _@MyAnnotation_ from _anotherMethod_ in _Bar_ class using the different utility methods

<pre class="lang:java decode:true ">Method barAnotherMethod = Bar.class.getMethod("anotherMethod");

System.out.println("getAnnotation (JDK): " + barAnotherMethod.getAnnotation(MyAnnotation.class));
System.out.println("getAnnotation (Spring): " + AnnotationUtils.getAnnotation(barAnotherMethod, MyAnnotation.class));
System.out.println("findAnnotation (Spring): " + AnnotationUtils.findAnnotation(barAnotherMethod, MyAnnotation.class));</pre>

will get the following results:

<div>
  <a style="margin-left: 1em; margin-right: 1em;" href="http://3.bp.blogspot.com/-mO2CLqb1GGk/VUZApkGv5lI/AAAAAAAAEkA/j-RvsfLoDYA/s1600/annot2.png"><img loading="lazy" class="alignnone" src="http://3.bp.blogspot.com/-mO2CLqb1GGk/VUZApkGv5lI/AAAAAAAAEkA/j-RvsfLoDYA/s1600/annot2.png" alt="findAnnotation result" width="478" height="50" border="0" /></a>
</div>

&nbsp;