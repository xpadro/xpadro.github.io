---
id: 33
title: Processing messages in transactions with Spring JMS
date: 2013-08-27T09:17:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/08/27/spring-jms-processing-messages-within-transactions/
permalink: /2013/08/spring-jms-processing-messages-within-transactions.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/08/spring-jms-processing-messages-within.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/8407595447198975915
categories:
  - Integration
  - Spring
  - Spring JMS
tags:
  - JMS
  - Spring
  - Test
---

This post shows how to process messages in transactions with Spring JMS. We will see how an error in the execution of the consumer during the asynchronous reception of messages with JMS, can lead to the loss of messages. I then will explain how you can solve this problem using local transactions.

You will also see that this solution can cause in some cases, message duplication (for example, when it saves the message to the database and then the listener execution fails). The reason why this happens is because the JMS transaction is independent to other transactional resources like the DB. If your processing is not idempotent or if your application does not support duplicate message detection, then you will have to use distributed transactions.

Distributed transactions are beyond the scope of this post. If you are interested in handling distributed transactions, you can read <a href="http://www.javaworld.com/javaworld/jw-01-2009/jw-01-spring-transactions.html?page=1" target="_blank" rel="noopener">Distributed transactions in Spring</a>.

&nbsp;

## <span style="color: #003366;">1 Use cases</span>

I&#8217;ve implemented a test application that reproduces the following cases:

<p style="padding-left: 30px;">
  1 Sending and reception of a message: The consumer will process the received message, storing it to a database.
</p>

<li style="list-style-type: none;">
  <ol>
    The producer sends the message to a queue:
  </ol>
</li>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://2.bp.blogspot.com/-JzRg50ba-Z8/VUZCUzx7diI/AAAAAAAAElo/TuBHuaHjpzw/s1600/send1.png"><img loading="lazy" class="alignnone" src="https://2.bp.blogspot.com/-JzRg50ba-Z8/VUZCUzx7diI/AAAAAAAAElo/TuBHuaHjpzw/s1600/send1.png" alt="send message diagram" width="320" height="41" border="0" /></a>
</div>

<div style="clear: both; text-align: center;">
</div>

<li style="list-style-type: none;">
  <ol>
    The consumer retrieves the message from the queue and processes it:
  </ol>
</li>

<div style="clear: both; text-align: center;">
  <a href="http://2.bp.blogspot.com/-2kryOM3PTuo/VUZCUmEMGvI/AAAAAAAAElc/UD7BEAT12Lw/s1600/send2.png"><img loading="lazy" class="alignnone" src="https://2.bp.blogspot.com/-2kryOM3PTuo/VUZCUmEMGvI/AAAAAAAAElc/UD7BEAT12Lw/s1600/send2.png" alt="receive message diagram" width="320" height="47" border="0" /></a>
</div>

<div>
</div>

<div style="clear: both; text-align: center;">
</div>

<p style="padding-left: 30px;">
  2 Error occurred before message processing: The consumer retrieves the message but the execution fails before storing it to the DB.
</p>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://1.bp.blogspot.com/-upkFvyyRXYI/VUZCUtAWnkI/AAAAAAAAElY/fzvI_PYWvCU/s1600/send3.png"><img loading="lazy" class="alignnone" src="https://1.bp.blogspot.com/-upkFvyyRXYI/VUZCUtAWnkI/AAAAAAAAElY/fzvI_PYWvCU/s1600/send3.png" alt="error before processing diagram" width="320" height="41" border="0" /></a>
</div>

<div>
</div>

<div style="clear: both; text-align: center;">
</div>

<p style="padding-left: 30px;">
  3 Error occurred after processing the message: The consumer retrieves the message, stores it to the DB, and then the execution fails.
</p>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://3.bp.blogspot.com/-vhgbj3FCPCk/VUZCUw_ftOI/AAAAAAAAElk/kntOwchqauo/s1600/send4.png"><img loading="lazy" class="alignnone" src="https://3.bp.blogspot.com/-vhgbj3FCPCk/VUZCUw_ftOI/AAAAAAAAElk/kntOwchqauo/s1600/send4.png" alt="error after processing diagram" width="320" height="40" border="0" /></a>
</div>

&nbsp;

The source code for this application can be found at <a href="https://github.com/xpadro/spring-integration/tree/master/spring-jms/jms-tx" target="_blank" rel="noopener">my Github repository</a>.

&nbsp;

## <span style="color: #003366;">2 The test application</span>

The test application executes two test classes, _TestNotTransactedMessaging_ and _TestTransactedMessaging_. These classes will both execute the three cases above described.

Let&#8217;s see the configuration of the application when it is executed without transactions.

&nbsp;

## <span style="color: #003366;">app-config.xml</span>

Application configuration. Basically it checks within the indicated packages to autodetect the application beans: producer and consumer. It also configures the in-memory database where processed notifications will be stored.

<pre class="lang:xhtml decode:true ">&lt;context:component-scan base-package="xpadro.spring.jms.producer, xpadro.spring.jms.receiver"/&gt;

&lt;bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate"&gt;
    &lt;constructor-arg ref="dataSource"/&gt;
&lt;/bean&gt;

&lt;jdbc:embedded-database id="dataSource"&gt;
    &lt;jdbc:script location="classpath:db/schema.sql" /&gt;
&lt;/jdbc:embedded-database&gt;</pre>

&nbsp;

## <span style="color: #003366;">notx-jms-config.xml</span>

Configures the JMS infrastructure, which is:

  * Broker connection
  * The JmsTemplate
  * Queue where notifications will be sent
  * The listener container that will send notifications to the listener to process them

<pre class="lang:xhtml decode:true ">&lt;!-- Infrastructure --&gt;
&lt;bean id="connectionFactory" class="org.apache.activemq.ActiveMQConnectionFactory"&gt;
    &lt;property name="brokerURL" value="vm://embedded?broker.persistent=false"/&gt;
&lt;/bean&gt;

&lt;bean id="cachingConnectionFactory" class="org.springframework.jms.connection.CachingConnectionFactory"&gt;
    &lt;property name="targetConnectionFactory" ref="connectionFactory"/&gt;
&lt;/bean&gt;

&lt;bean id="jmsTemplate" class="org.springframework.jms.core.JmsTemplate"&gt;
    &lt;property name="connectionFactory" ref="cachingConnectionFactory"/&gt;
    &lt;property name="defaultDestination" ref="incomingQueue"/&gt;
&lt;/bean&gt;

&lt;!-- Destinations --&gt;
&lt;bean id="incomingQueue" class="org.apache.activemq.command.ActiveMQQueue"&gt;
    &lt;constructor-arg value="incoming.queue"/&gt;
&lt;/bean&gt;
	
&lt;!-- Listeners --&gt;
&lt;jms:listener-container connection-factory="connectionFactory"&gt;
    &lt;jms:listener ref="notificationProcessor" destination="incoming.queue"/&gt;
&lt;/jms:listener-container&gt;</pre>

&nbsp;

The producer simply uses the jmsTemplate to send notifications:

<pre class="lang:java decode:true ">@Component("producer")
public class Producer {
    private static Logger logger = LoggerFactory.getLogger(Producer.class);
    
    @Autowired
    private JmsTemplate jmsTemplate;
    
    public void convertAndSendMessage(String destination, Notification notification) {
        jmsTemplate.convertAndSend(destination, notification);
        logger.info("Sending notification | Id: "+notification.getId());
    }
}</pre>

&nbsp;

The listener is responsible for retrieval of notifications from the queue and stores them to the database:

<pre class="lang:java decode:true ">@Component("notificationProcessor")
public class NotificationProcessor implements MessageListener {
    private static Logger logger = LoggerFactory.getLogger(NotificationProcessor.class);
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Override
    public void onMessage(Message message) {
        try {
            Notification notification = (Notification) ((ObjectMessage) message).getObject();
            logger.info("Received notification | Id: "+notification.getId()+" | Redelivery: "+getDeliveryNumber(message));
            
            checkPreprocessException(notification);
            saveToBD(notification);
            checkPostprocessException(message, notification);
        } catch (JMSException e) {
            throw JmsUtils.convertJmsAccessException(e);
        }
    }	
    ...
}</pre>

The _checkPreprocessException_ method will throw a runtime exception when a notification with id=1 arrive. In this way, we will cause an error before storing the message to the DB.

The _checkPostprocessException_ method will throw an exception if a notification with id=2 arrive, thereby causing an error just after storing it to the DB.

The _getDeliveryNumber_ method returns the number of times the message has been sent. This only applies within transactions, since the broker will try to resend the message after listener processing failure led to a rollback.

Finally, the _saveToDB_ method is pretty obvious. It stores a notification to the DB.

You can always check the source code of this application by clicking the link at the beginning of this article.

&nbsp;

## <span style="color: #003366;">3 Testing message reception without transactions</span>

I will launch two test classes, one without transactions and the other within a local transaction. Both classes extend a base class that loads the common application context and contains some utility methods:

<pre class="lang:default decode:true ">@ContextConfiguration(locations = {"/xpadro/spring/jms/config/app-config.xml"})
@DirtiesContext
public class TestBaseMessaging {
    protected static final String QUEUE_INCOMING = "incoming.queue";
    protected static final String QUEUE_DLQ = "ActiveMQ.DLQ";
    
    @Autowired
    protected JdbcTemplate jdbcTemplate;
    
    @Autowired
    protected JmsTemplate jmsTemplate;
    
    @Autowired
    protected Producer producer;
    
    @Before
    public void prepareTest() {
        jdbcTemplate.update("delete from Notifications");
    }
    
    protected int getSavedNotifications() {
        return jdbcTemplate.queryForObject("select count(*) from Notifications", Integer.class);
    }
    
    protected int getMessagesInQueue(String queueName) {
        return jmsTemplate.browse(queueName, new BrowserCallback&lt;Integer&gt;() {
            @Override
            public Integer doInJms(Session session, QueueBrowser browser) throws JMSException {
                Enumeration&lt;?&gt; messages = browser.getEnumeration();
                int total = 0;
                while (messages.hasMoreElements()) {
                    messages.nextElement();
                    total++;
                }
                
                return total;
            }
        });
    }
}</pre>

&nbsp;

The utility methods are explained below:

  * _getSavedNotifications_: Returns the number of notifications stored to the DB. I&#8217;ve used the queryForObject method because it is the recommended since version 3.2.2. The queryForInt method has been deprecated.
  * _getMessagesInQueue_: Allows you to check which messages are still pending in the specified queue. For this test we are interested in knowing how many notifications are still waiting to be processed.

Now, let me show you the code for the first test (_TestNotTransactedMessaging_). This test launches the 3 cases indicated at the beginning of the article:

<pre class="lang:java decode:true">@Test
public void testCorrectMessage() throws InterruptedException {
    Notification notification = new Notification(0, "notification to deliver correctly");
    producer.convertAndSendMessage(QUEUE_INCOMING, notification);
    
    Thread.sleep(6000);
    printResults();
    
    assertEquals(1, getSavedNotifications());
    assertEquals(0, getMessagesInQueue(QUEUE_INCOMING));
}

@Test
public void testFailedAfterReceiveMessage() throws InterruptedException {
    Notification notification = new Notification(1, "notification to fail after receiving");
    producer.convertAndSendMessage(QUEUE_INCOMING, notification);
    
    Thread.sleep(6000);
    printResults();
    
    assertEquals(0, getSavedNotifications());
    assertEquals(0, getMessagesInQueue(QUEUE_INCOMING));
}

@Test
public void testFailedAfterProcessingMessage() throws InterruptedException {
    Notification notification = new Notification(2, "notification to fail after processing");
    producer.convertAndSendMessage(QUEUE_INCOMING, notification);
    
    Thread.sleep(6000);
    printResults();
    
    assertEquals(1, getSavedNotifications());
    assertEquals(0, getMessagesInQueue(QUEUE_INCOMING));
}

private void printResults() {
    logger.info("Total items in \"incoming\" queue: "+getMessagesInQueue(QUEUE_INCOMING));
    logger.info("Total items in DB: "+getSavedNotifications());
}</pre>

&nbsp;

## <span style="color: #003366;">4 Executing the test</span>

Ok, let&#8217;s execute the test and see what the results are:

**testCorrectMessage output:**

<div style="font-size: x-small;">
  Producer|Sending notification | Id: 0<br /> NotificationProcessor|Received notification | Id: 0 | Redelivery: 1<br /> TestNotTransactedMessaging|Total items in &#8220;incoming&#8221; queue: 0<br /> TestNotTransactedMessaging|Total items in DB: 1
</div>

No problem here, the queue is empty since the message has been correctly received and stored to the database.

**testFailedAfterReceiveMessage output:**

<div style="font-size: x-small;">
  Producer|Sending notification | Id: 1<br /> NotificationProcessor|Received notification | Id: 1 | Redelivery: 1<br /> AbstractMessageListenerContainer|Execution of JMS message listener failed, and no ErrorHandler has been set.<br /> java.lang.RuntimeException: error after receiving message<br /> TestNotTransactedMessaging|Total items in &#8220;incoming&#8221; queue: 0<br /> TestNotTransactedMessaging|Total items in DB: 0
</div>

Since it is executing outside a transaction, the acknowledge mode (auto by default) is used. This implies that the message is considered successfully delivered once the onMessage method is invoked and therefore deleted from the queue. Because the listener failed before storing the message to the DB, we have lost the message!!

**testFailedAfterProcessingMessage output:**

<div style="font-size: x-small;">
  2013-08-22 18:39:09,906|Producer|Sending notification | Id: 2<br /> 2013-08-22 18:39:09,906|NotificationProcessor|Received notification | Id: 2 | Redelivery: 1<br /> 2013-08-22 18:39:09,906|AbstractMessageListenerContainer|Execution of JMS message listener failed, and no ErrorHandler has been set.<br /> java.lang.RuntimeException: error after processing message<br /> 2013-08-22 18:39:15,921|TestNotTransactedMessaging|Total items in &#8220;incoming&#8221; queue: 0<br /> 2013-08-22 18:39:15,921|TestNotTransactedMessaging|Total items in DB: 1
</div>

In this case, the message has been deleted from the queue (AUTO_ACKNOWLEDGE) and stored to the DB before the execution failed.

&nbsp;

## <span style="color: #003366;">5 Adding local transactions</span>

Usually we can&#8217;t allow losing messages like the second case of the test, so what we will do is to invoke the listener within a local transaction. The change is pretty simple and it does not imply modifying a single line of code from our application. We will only need to change the configuration file.

To test the 3 cases with transactions, I will replace the configuration file _notx-jms-config.xml_ for the following:

### <span style="color: #003366;">tx-jms-config.xml</span>

First, I&#8217;ve added the number of re-deliveries made in case of a rollback (caused by an error in the listener execution):

<pre class="lang:xhtml decode:true ">&lt;bean id="connectionFactory" class="org.apache.activemq.ActiveMQConnectionFactory"&gt;
    &lt;property name="brokerURL" value="vm://embedded?broker.persistent=false"/&gt;
    &lt;property name="redeliveryPolicy"&gt;
        &lt;bean class="org.apache.activemq.RedeliveryPolicy"&gt;
            &lt;property name="maximumRedeliveries" value="4"/&gt;
        &lt;/bean&gt;
    &lt;/property&gt;
&lt;/bean&gt;</pre>

&nbsp;

Next, I indicate that the listener will be executed within a transaction. This can be done by modifying the listener container definition:

<pre class="lang:xhtml decode:true ">&lt;jms:listener-container connection-factory="connectionFactory" acknowledge="transacted"&gt;
    &lt;jms:listener ref="notificationProcessor" destination="incoming.queue"/&gt;
&lt;/jms:listener-container&gt;</pre>

&nbsp;

This will cause every invocation of the listener to be executed within a local JMS transaction. The transaction will start when the message is received. If the listener execution fails, message reception will be rolled back.

And that&#8217;s all we have to change. Let&#8217;s launch the tests with this configuration.

&nbsp;

## <span style="color: #003366;">6 Testing message reception within transactions</span>

The code from the TestTransactedMessaging class is practically the same as the previous test. The only difference is that it adds a query to the DLQ (dead letter queue). When executed within transactions, if the message reception is rolled back, the broker will send the message to this queue (after all re-deliveries failed).

I&#8217;m skipping the output of the successful receiving as it does not bring anything new.

**testFailedAfterReceiveMessage output:**

<div style="font-size: x-small;">
  Producer|Sending notification | Id: 1<br /> NotificationProcessor|Received notification | Id: 1 | Redelivery: 1<br /> AbstractMessageListenerContainer|Execution of JMS message listener failed, and no ErrorHandler has been set.<br /> java.lang.RuntimeException: error after receiving message<br /> NotificationProcessor|Received notification | Id: 1 | Redelivery: 2<br /> AbstractMessageListenerContainer|Execution of JMS message listener failed, and no ErrorHandler has been set.<br /> &#8230;<br /> java.lang.RuntimeException: error after receiving message<br /> NotificationProcessor|Received notification | Id: 1 | Redelivery: 5<br /> AbstractMessageListenerContainer|Execution of JMS message listener failed, and no ErrorHandler has been set.<br /> java.lang.RuntimeException: error after receiving message<br /> TestTransactedMessaging|Total items in &#8220;incoming&#8221; queue: 0<br /> TestTransactedMessaging|Total items in &#8220;dead letter&#8221; queue: 1<br /> TestTransactedMessaging|Total items in DB: 0
</div>

As you can see, the first receiving has failed, and the broker has tried to resend it four more times (as indicated in the maximumRedeliveries property). Since the situation persisted, the message has been sent to the special DLQ queue. In this way, we do not lose the message.

**testFailedAfterProcessingMessage output:**

<div style="font-size: x-small;">
  Producer|Sending notification | Id: 2<br /> NotificationProcessor|Received notification | Id: 2 | Redelivery: 1<br /> AbstractMessageListenerContainer|Execution of JMS message listener failed, and no ErrorHandler has been set.<br /> java.lang.RuntimeException: error after processing message<br /> NotificationProcessor|Received notification | Id: 2 | Redelivery: 2<br /> TestTransactedMessaging|Total items in &#8220;incoming&#8221; queue: 0<br /> TestTransactedMessaging|Total items in &#8220;dead letter&#8221; queue: 0<br /> TestTransactedMessaging|Total items in DB: 2
</div>

In this case, this is what happened:

  1. The listener retrieved the message
  2. It stored the message to the DB
  3. Listener execution failed
  4. The broker resends the message. Since the situation has been solved, the listener stores the message to the DB (again). The message has been duplicated.

&nbsp;

## <span style="color: #003366;">7 Conclusion</span>

Adding local transactions to the message reception avoids losing messages. What we have to take into account is that duplicate messages can occur, so our listener will have to detect it, or our processing will have to be idempotent to process it again without any problem. If this is not possible, we will have to go for distributed transactions, since they support transactions that involve different resources.

&nbsp;