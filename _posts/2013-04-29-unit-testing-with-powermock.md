---
id: 38
title: Unit testing with PowerMock
date: 2013-04-29T17:51:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/04/29/unit-testing-with-powermock/
permalink: /2013/04/unit-testing-with-powermock.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/04/unit-testing-with-powermock.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/2284417266019754194
categories:
  - Java
  - Test
tags:
  - PowerMock
  - Test
---
In this article I will implement unit testing with PowerMock library. This framework is more powerful than other libraries and allows you to mock static methods, private methods and change field properties among other things. I will use the Mockito extension but it also supports EasyMock.

## <span style="color: #0b5394;">1   Installation</span>

If you use Maven, you can add the following dependencies to your pom.xml:

<pre class="lang:xhtml decode:true ">&lt;dependency&gt;
    &lt;groupId&gt;org.powermock&lt;/groupId&gt;
    &lt;artifactId&gt;powermock-api-mockito&lt;/artifactId&gt;
    &lt;version&gt;1.5&lt;/version&gt;
    &lt;scope&gt;test&lt;/scope&gt;
&lt;/dependency&gt;
&lt;dependency&gt;
    &lt;groupId&gt;org.powermock&lt;/groupId&gt;
    &lt;artifactId&gt;powermock-module-junit4&lt;/artifactId&gt;
    &lt;version&gt;1.5&lt;/version&gt;
    &lt;scope&gt;test&lt;/scope&gt;
&lt;/dependency&gt;</pre>

Otherwise, you can get the libraries from the <a href="http://code.google.com/p/powermock/downloads/list" target="_blank" rel="noopener">powermock github repository</a>.

## <span style="color: #0b5394;">2   Preparing the tests</span>

You must add the @RunWith annotation. This annotation will tell JUnit to use the class PowerMockRunner to run the tests.

<pre class="lang:java decode:true ">@RunWith(PowerMockRunner.class)
public class FooTester {
    private Foo foo;
    
    @Before
    public void prepareTest() {
        foo = PowerMockito.spy(new Foo());
    }</pre>

I&#8217;m going to test Foo.class. Using the _spy_ method, all real methods of the class will be invoked except those I mock.

## <span style="color: #0b5394;">3   Mocking private methods</span>

I want to test _methodWithPrivateDependency_, but this method internally calls a private method which, let&#8217;s say, calls an external resource.

<pre class="lang:java decode:true ">public String methodWithPrivateDependency() {
    myPrivateMethod();
    return resolveApplicationId();
}
	
private String resolveApplicationId() {
    return "testApplication";
}
	
private void myPrivateMethod() {
    throw new RuntimeException("tsk, tsk you invoked private method!");
}</pre>

The test:

<pre class="lang:java decode:true ">@Test
@PrepareForTest(Foo.class)
public void checkApplicationIdIsResolved() throws Exception {
    PowerMockito.doNothing().when(foo, "myPrivateMethod");
    Assert.assertEquals("testApplication", foo.methodWithPrivateDependency());
    PowerMockito.verifyPrivate(foo).invoke("myPrivateMethod");
}</pre>

The @PrepareForTest annotation tells PowerMock to prepare the class so it can manipulate it. This is necessary if the class is final, or it contains final, private, static or native methods that should be mocked. In this case, I will need to mock the private method.

If you put this annotation at class level, it will affect all test methods within the class. It would be as follows:

<pre class="lang:java decode:true">@PrepareForTest({Class1.class, ..., ClassN.class})</pre>

&nbsp;

## <span style="color: #0b5394;">4   Mocking static access</span>

The next method that needs to be tested gets some information by accessing a static method of an inner static class. Seems like we need to mock the static access.

The method to test:

<pre class="lang:java decode:true ">public String getCriticalMessage() {
    String name = Bar.Baz.getData();
    return "hello " + name;
}</pre>

&nbsp;

The static class:

<pre class="lang:java decode:true ">public class Bar {
    public static class Baz {
        public static String getData() {
            throw new RuntimeException("tsk, tsk you invoked getData!");
        }
    }
}</pre>

The test:

<pre class="lang:java decode:true ">@Test
@PrepareForTest(Bar.Baz.class)
public void testStaticDependency() {
    PowerMockito.mockStatic(Bar.Baz.class);
    PowerMockito.when(Bar.Baz.getData()).thenReturn("world!");
    Assert.assertEquals("hello world!", foo.getCriticalMessage());
}</pre>

&nbsp;

I use the @PrepareForTest annotation because I need to mock the static method _getData_ of this class.

## <span style="color: #0b5394;">5   Mocking chained dependencies</span>

In this example, I want to mock the _chainMocks_ method. There&#8217;s a singleton _Helper_ class which returns a manager from which the method to test invokes it.

<pre class="lang:java decode:true ">public String chainMocks(){
    Helper.getInstance().getConfigurationManager().initializeRegistry();
    return "chained";
}</pre>

The test:

<pre class="lang:java decode:true ">@Test
@PrepareForTest(Helper.class)
public void testChainedDependencies() {
    Helper helperMock = PowerMockito.mock(Helper.class);
    PowerMockito.mockStatic(Helper.class);
    PowerMockito.when(Helper.getInstance()).thenReturn(helperMock);
    
    ConfigurationManager mockManager = PowerMockito.mock(ConfigurationManager.class);
    PowerMockito.when(helperMock.getConfigurationManager()).thenReturn(mockManager);
    
    Assert.assertEquals("chained", foo.chainMocks());
}</pre>

I only need to prepare _Helper_ class for test because I need to mock its static method _getInstance._ That&#8217;s not the case for _ConfigurationManager_ where _initializeRegistry_ is a public instance method.