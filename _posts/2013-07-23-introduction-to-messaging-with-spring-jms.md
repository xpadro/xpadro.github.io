---
id: 34
title: Introduction to messaging with Spring JMS
date: 2013-07-23T16:30:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/07/23/introduction-to-messaging-with-spring-jms/
permalink: /2013/07/introduction-to-messaging-with-spring-jms.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/07/introduction-to-messaging-with-spring.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/6043040076910427392
categories:
  - Integration
  - Spring
  - Spring JMS
tags:
  - JMS
  - Spring
  - Test
---

In this post I will show you how to configure a standalone application in order to see different ways of sending and receiving messages using Spring JMS. Basically, I will divide the examples into the following sections:

  * Point-to-point messaging (queue) 
      * Synchronous reception
      * Asynchronous reception
  * Publish-subscribe messaging (topic)

The source code with all the examples shown in this article is available at <a href="https://github.com/xpadro/spring-integration/tree/master/spring-jms/jms-basic" target="_blank" rel="noopener">my Github repository</a>.

## <span style="color: #0b5394;">1 Configuring the provider</span>

The first thing we need to do is to configure the ConnectionFactory. The connection factory is part of the JMS specification and allows the application to create connections with the JMS provider:

<pre class="lang:xhtml decode:true ">&lt;bean id="connectionFactory" class="org.apache.activemq.ActiveMQConnectionFactory"&gt;
    &lt;property name="brokerURL" value="vm://embedded?broker.persistent=false"/&gt;
&lt;/bean&gt;</pre>

The factory is used to create a connection which is then used to create a session. In the following examples, we won&#8217;t need to care about this since the JmsTemplate class will do this for us.

Spring provides its own ConnectionFactory implementations, which are specified below:

  * SingleConnectionFactory: All _createConnection()_ calls will return the same connection. This is useful for testing.
  * CachingConnectionFactory: It provides caching of sessions.

The JmsTemplate aggressively opens and closes resources like sessions since it assumes that are cached by the connectionFactory. Using the CachingConnectionFactory will improve its performance. In our example, we will define a cachingConnectionFactory passing our previously defined AMQ connectionFactory to its _targetConnectionFactory_ property:

<pre class="lang:xhtml decode:true">&lt;bean id="cachingConnectionFactory" class="org.springframework.jms.connection.CachingConnectionFactory"&gt;
    &lt;property name="targetConnectionFactory" ref="connectionFactory"/&gt;
&lt;/bean&gt;
</pre>

&nbsp;

## <span style="color: #0b5394;">2 Point-to-point messaging (queue)</span>

This Destination implementation consists in sending a message to a single consumer. The producer will send a message to the queue where it will be retrieved by the consumer.

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://1.bp.blogspot.com/-XGiBLspp7gw/VUZB115xUvI/AAAAAAAAElA/2nY9p1PhDAE/s1600/queue.png"><img loading="lazy" class="alignnone" src="https://1.bp.blogspot.com/-XGiBLspp7gw/VUZB115xUvI/AAAAAAAAElA/2nY9p1PhDAE/s1600/queue.png" alt="point-to-point diagram" width="400" height="106" border="0" /></a>
</div>

&nbsp;

The consumer will actively retrieve the message from the queue (synchronous reception) or it will retrieve the message passively (asynchronous reception). Now we will see an example of each.

### <span style="color: #0b5394;">2.1 Synchronous reception</span>

#### <span style="color: #0b5394;">2.1.1 Configuration</span>

Spring JMS uses JmsTemplate class for message production and synchronous message reception. This template is a central class of Spring JMS, and helps us by:

  * Reducing boilerplate code: It handles the creation and release of resources transparently (connection, session&#8230;).
  * Handling exceptions and converting them to runtime exceptions.
  * Providing utility methods and callbacks.

Let&#8217;s configure the jmsTemplate:

<pre class="lang:xhtml decode:true ">&lt;bean id="jmsTemplate" class="org.springframework.jms.core.JmsTemplate"&gt;
    &lt;property name="connectionFactory" ref="cachingConnectionFactory"/&gt;
    &lt;property name="defaultDestination" ref="syncTestQueue"/&gt;
&lt;/bean&gt;</pre>

This template will handle our point-to-point messaging. To use topics it will need further configuration, which will be shown in the following sections.

The Queue destination is defined below:

<pre class="lang:xhtml decode:true ">&lt;bean id="syncTestQueue" class="org.apache.activemq.command.ActiveMQQueue"&gt;
    &lt;constructor-arg value="test.sync.queue"/&gt;
&lt;/bean&gt;</pre>

In our example, a producer will send a message to this queue and a consumer will retrieve it.

#### <span style="color: #0b5394;">2.1.2 The producer</span>

<pre class="lang:java decode:true ">@Component("producer")
public class Producer {
    @Autowired
    @Qualifier("jmsTemplate")
    private JmsTemplate jmsTemplate;
    
    public void convertAndSendMessage(Notification notification) {
        jmsTemplate.convertAndSend(notification);
    }
</pre>

&nbsp;

This producer will use a jmsTemplate to send a message. Note the @Component annotation, this class will be auto detected and registered as a bean.

It is also important to see that we are passing a Notification object to the jmsTemplate method. If we do not define a message converter, the template will register a SimpleMessageConverter by default (check JmsTemplate constructor). This converter will be able to convert the following types:

  * String to TextMessage
  * Serializable to ObjectMessage
  * Map to MapMessage
  * byte[] to BytesMessage

If the object being sent is not an instance of any of the previous list, it will throw a MessageConversionException. The common cause of this exception is that your object is not implementing Serializable interface.

In this case, it will convert our Notification object to an ObjectMessage and send it to its default destination, which we previously defined as &#8220;_test.sync.queue_&#8220;.

#### <span style="color: #0b5394;">2.1.3 The consumer</span>

<pre class="lang:java decode:true ">@Component
public class SyncReceiver {
    @Autowired
    private JmsTemplate jmsTemplate;
    
    public Notification receive() {
        return (Notification) jmsTemplate.receiveAndConvert("test.sync.queue");
    }
}</pre>

You should use this method carefully as it blocks the current thread until it receives the message. You should better define a timeout in case there&#8217;s a problem receiving the message. The jmsTemplate has no timeout setter method defined. You will need to define it when configuring the AMQ connection factory:

<div style="text-align: center;">
  <property name=&#8221;sendTimeout&#8221; value=&#8221;5000&#8243;/>
</div>

#### <span style="color: #0b5394;">2.1.4 The test</span>

<pre class="lang:java decode:true ">@ContextConfiguration(locations = {
    "/xpadro/spring/jms/config/jms-config.xml", 
    "/xpadro/spring/jms/config/app-config.xml"})
@RunWith(SpringJUnit4ClassRunner.class)
public class TestSyncMessaging {
    
    @Autowired
    private Producer producer;
    
    @Autowired
    private SyncReceiver syncReceiver;
    
    @Test
    public void testSynchronizedReceiving() throws InterruptedException {
        Notification notification = new Notification("1", "this is a message");
        //Sends the message to the jmsTemplate's default destination
        producer.convertAndSendMessage(notification);
        Thread.sleep(2000);
        
        Notification receivedNotification = syncReceiver.receive();
        assertNotNull(receivedNotification);
        assertEquals("this is a message", receivedNotification.getMessage());
    }
}</pre>

&nbsp;

### <span style="color: #0b5394;">2.2 Asynchronous reception</span>

Spring lets you receive messages asynchronously in two different ways:

  * Implementing MessageListener interface
  * Using a simple POJO

The following example will show the second approach.

#### <span style="color: #0b5394;">2.2.1 Configuration</span>

We can use the same jmsTemplate we configured in the previous example. In this case we will configure another queue where the producer will send its message and a consumer that will act as a listener:

<pre class="lang:xhtml decode:true ">&lt;bean id="asyncTestQueue" class="org.apache.activemq.command.ActiveMQQueue"&gt;
    &lt;constructor-arg value="test.async.queue"/&gt;
&lt;/bean&gt;

&lt;jms:listener-container connection-factory="connectionFactory"&gt;
    &lt;jms:listener destination="test.async.queue" ref="asyncReceiver" method="receiveMessage"/&gt;
&lt;/jms:listener-container&gt;</pre>

&nbsp;

We are configuring a consumer which will be the _asyncReceiver_ bean, and the listener container will invoke its _receiveMessage_ method when a message arrives to the _test.async.queue_.

#### <span style="color: #0b5394;">2.2.2 The producer</span>

The producer will be the same defined in the previous section, but it will send the message to a different queue:

<pre class="lang:java decode:true ">public void convertAndSendMessage(String destination, Notification notification) {
    jmsTemplate.convertAndSend(destination, notification);
}</pre>

#### <span style="color: #0b5394;">2.2.3 The consumer</span>

<pre class="lang:java decode:true">@Component("asyncReceiver")
public class AsyncReceiver {
    @Autowired
    private NotificationRegistry registry;
    
    public void receiveMessage(Notification notification) {
        registry.registerNotification(notification);
    }
}</pre>

&nbsp;

As you can see, it&#8217;s a simple Java class. It does not need to implement any interface. The consumer saves received notifications to a registry. This registry will be used by the test class to assert that notifications arrived correctly.

#### <span style="color: #0b5394;">2.2.4 The test</span>

<pre class="lang:java decode:true ">@ContextConfiguration(locations = {
    "/xpadro/spring/jms/config/jms-config.xml", 
    "/xpadro/spring/jms/config/app-config.xml"})
@RunWith(SpringJUnit4ClassRunner.class)
public class TestAsyncMessaging {
    
    @Autowired
    private Producer producer;
    
    @Autowired
    private NotificationRegistry registry;
    
    @Test
    public void testAsynchronizedReceiving() throws InterruptedException {
        Notification notification = new Notification("2", "this is another message");
        producer.convertAndSendMessage("test.async.queue", notification);
        Thread.sleep(2000);
        
        assertEquals(1, registry.getReceivedNotifications().size());
        assertEquals("this is another message", registry.getReceivedNotifications().get(0).getMessage());
    }
}</pre>

&nbsp;

## <span style="color: #0b5394;">3 Publish-subscribe messaging (topic)</span>

The message is sent to a topic, where it will be distributed to all consumers that are subscribed to this topic.

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://1.bp.blogspot.com/-Noex8BP_5XA/VUZB10tvp6I/AAAAAAAAEk8/kdaL7-pDe1Y/s1600/topic.png"><img loading="lazy" class="alignnone" src="https://1.bp.blogspot.com/-Noex8BP_5XA/VUZB10tvp6I/AAAAAAAAEk8/kdaL7-pDe1Y/s1600/topic.png" alt="publish-subscribe diagram" width="400" height="106" border="0" /></a>
</div>

&nbsp;

### <span style="color: #0b5394;">3.1 Configuration</span>

We will need another jmsTemplate since the template we configured before is set to work with queues.

<pre class="lang:xhtml decode:true ">&lt;bean id="jmsTopicTemplate" class="org.springframework.jms.core.JmsTemplate"&gt;
    &lt;property name="connectionFactory" ref="cachingConnectionFactory"/&gt;
    &lt;property name="pubSubDomain" value="true"/&gt;
&lt;/bean&gt;</pre>

We just need to configure its destination accessor by defining its _pubSubDomain_ property and set its value to true. The default value is false (point-to-point).

Next, we configure a new destination, which will be the topic for this example:

<pre class="lang:xhtml decode:true ">&lt;bean id="testTopic" class="org.apache.activemq.command.ActiveMQTopic"&gt;
    &lt;constructor-arg value="test.topic"/&gt;
&lt;/bean&gt;</pre>

Finally, we define the listeners. We define two listeners to make sure both consumers receive the message sent to the topic by the producer.

<pre class="lang:xhtml decode:true ">&lt;jms:listener-container connection-factory="connectionFactory" destination-type="topic"&gt;
    &lt;jms:listener destination="test.topic" ref="asyncTopicFooReceiver" method="receive"/&gt;
    &lt;jms:listener destination="test.topic" ref="asyncTopicBarReceiver" method="receive"/&gt;
&lt;/jms:listener-container&gt;</pre>

You may notice a difference in the listener configuration. We need to change the destination type of the listener container, which is set to queue by default. Just set its value to topic and we are done.

### <span style="color: #0b5394;">3.2 The producer</span>

<pre class="lang:java decode:true">@Component("producer")
public class Producer {
    @Autowired
    @Qualifier("jmsTopicTemplate")
    private JmsTemplate jmsTopicTemplate;
    
    public void convertAndSendTopic(Notification notification) {
        jmsTopicTemplate.convertAndSend("test.topic", notification);
    }
}</pre>

### <span style="color: #0b5394;">3.3 The consumer</span>

<pre class="lang:java decode:true ">@Component("asyncTopicBarReceiver")
public class AsyncTopicBarReceiver {
    @Autowired
    private NotificationRegistry registry;
    
    public void receive(Notification notification) {
        registry.registerNotification(notification);
    }
}</pre>

The _asyncTopicFooReceiver_ has the same method.

### <span style="color: #0b5394;">3.4 The test</span>

<pre class="lang:java decode:true ">@ContextConfiguration(locations = {
    "/xpadro/spring/jms/config/jms-config.xml", 
    "/xpadro/spring/jms/config/app-config.xml"})
@RunWith(SpringJUnit4ClassRunner.class)
public class TestTopicMessaging {
    
    @Autowired
    private Producer producer;
    
    @Autowired
    private NotificationRegistry registry;
    
    @Test
    public void testTopicSending() throws InterruptedException {
        Notification notification = new Notification("3", "this is a topic");
        producer.convertAndSendTopic(notification);
        Thread.sleep(2000);
        
        assertEquals(2, registry.getReceivedNotifications().size());
        assertEquals("this is a topic", registry.getReceivedNotifications().get(0).getMessage());
        assertEquals("this is a topic", registry.getReceivedNotifications().get(1).getMessage());
    }
}</pre>

&nbsp;