---
id: 14
title: Configure a Spring JMS application with Spring Boot and annotation support
date: 2015-04-14T17:12:00+01:00
author: xpadro
layout: post
permalink: /2015/04/configure-a-spring-jms-application-with-spring-boot-and-annotation-support.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2015/04/configure-spring-jms-application-with.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/7402829438683298298
image: /wp-content/uploads/2015/04/20150414_1.png
categories:
  - Spring
  - Spring JMS
  - Spring-Boot
tags:
  - JMS
  - Spring
  - Spring Boot
---

In previous posts we learned how to configure a project using Spring JMS. If you check the article <a href="http://xpadro.com/2013/07/introduction-to-messaging-with-spring.html" target="_blank" rel="noopener">introduction to messaging with Spring JMS</a>, you will notice that it is configured using XML. This article will take advantage of the <a href="https://spring.io/blog/2014/04/30/spring-4-1-s-upcoming-jms-improvements" target="_blank" rel="noopener">improvements</a> introduced in Spring 4.1 version, and configure a Spring JMS application with Spring Boot using Java config only.

In this example we will also see how easy it can be to configure the project by using <a href="http://projects.spring.io/spring-boot/" target="_blank" rel="noopener">Spring Boot</a>.

Before we get started, just note that as usual, you can take a look at the source code of the project used in the examples below.

<a href="https://github.com/xpadro/spring-integration/tree/master/spring-jms/jms-boot-javaconfig" target="_blank" rel="noopener">See the example project at github</a>.

Sections:

  1. Introduction.
  2. The example application.
  3. Setting up the project.
  4. A simple example with JMS listener.
  5. Sending a response to another queue with @SendTo.
  6. Conclusion.

&nbsp;

## <span style="color: #0b5394;">1 The example application</span>

The application uses a Client service to send orders to a JMS queue, where a JMS listener will be registered and handle these orders. Once received, the listener will store the order through the Store service:

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <img loading="lazy" class="alignnone wp-image-204 size-full" src="http://xpadro.com/wp-content/uploads/2015/04/jmsDiagram.png" alt="Spring JMS application with Spring Boot diagram" width="655" height="382" srcset="https://xpadro.com/wp-content/uploads/2015/04/jmsDiagram.png 655w, https://xpadro.com/wp-content/uploads/2015/04/jmsDiagram-300x175.png 300w" sizes="(max-width: 655px) 100vw, 655px" />
</div>

&nbsp;

We will use the Order class to create orders:

<pre class="lang:java decode:true ">public class Order implements Serializable {
    private static final long serialVersionUID = -797586847427389162L;
    
    private final String id;
    
    public Order(String id) {
        this.id = id;
    }
    
    public String getId() {
        return id;
    }
}</pre>

&nbsp;

Before moving on to the first example, we will first explore how the project structure is built.

## <span style="color: #0b5394;">2 Setting up the project</span>

### <span style="color: #0b5394;">2.1 Configuring pom.xml</span>

The first thing to do is to define the artifact _spring-boot-starter-parent_ as our parent pom.

<pre class="lang:xhtml decode:true ">&lt;parent&gt;
    &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
    &lt;artifactId&gt;spring-boot-starter-parent&lt;/artifactId&gt;
    &lt;version&gt;1.2.3.RELEASE&lt;/version&gt;
    &lt;relativePath/&gt; &lt;!-- lookup parent from repository --&gt;
&lt;/parent&gt;</pre>

&nbsp;

This parent basically sets several Maven defaults and provides the dependency management for the main dependencies that we will use, like the Spring version (which is 4.1.6).

It is important to note that this parent pom defines the version of many libraries but it does not add any dependency to our project. So don’t worry about getting libraries you won’t use.

The next step is to set the basic dependencies for Spring Boot:

<pre class="lang:xhtml decode:true ">&lt;dependency&gt;
    &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
    &lt;artifactId&gt;spring-boot-starter&lt;/artifactId&gt;
&lt;/dependency&gt;</pre>

&nbsp;

In addition to the core Spring libraries, this dependency will bring the auto configuration functionality of Spring Boot. This will allow the framework to try to automatically set up the configuration based on the dependencies you add.

Finally, we will add the Spring JMS dependency and the ActiveMQ message broker, leaving the whole pom.xml as follows:

<pre class="lang:xhtml decode:true">&lt;groupId&gt;xpadro.spring&lt;/groupId&gt;
&lt;artifactId&gt;jms-boot-javaconfig&lt;/artifactId&gt;
&lt;version&gt;0.0.1-SNAPSHOT&lt;/version&gt;
&lt;packaging&gt;jar&lt;/packaging&gt;
&lt;name&gt;JMS Spring Boot Javaconfig&lt;/name&gt;

&lt;parent&gt;
    &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
    &lt;artifactId&gt;spring-boot-starter-parent&lt;/artifactId&gt;
    &lt;version&gt;1.2.3.RELEASE&lt;/version&gt;
    &lt;relativePath/&gt; &lt;!-- lookup parent from repository --&gt;
&lt;/parent&gt;

&lt;properties&gt;
    &lt;project.build.sourceEncoding&gt;UTF-8&lt;/project.build.sourceEncoding&gt;
    &lt;start-class&gt;xpadro.spring.jms.JmsJavaconfigApplication&lt;/start-class&gt;
    &lt;java.version&gt;1.8&lt;/java.version&gt;
    &lt;amq.version&gt;5.4.2&lt;/amq.version&gt;
&lt;/properties&gt;

&lt;dependencies&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
        &lt;artifactId&gt;spring-boot-starter&lt;/artifactId&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework&lt;/groupId&gt;
        &lt;artifactId&gt;spring-jms&lt;/artifactId&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
        &lt;artifactId&gt;spring-boot-starter-test&lt;/artifactId&gt;
        &lt;scope&gt;test&lt;/scope&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.apache.activemq&lt;/groupId&gt;
        &lt;artifactId&gt;activemq-core&lt;/artifactId&gt;
        &lt;version&gt;${amq.version}&lt;/version&gt;
    &lt;/dependency&gt;
&lt;/dependencies&gt;

&lt;build&gt;
    &lt;plugins&gt;
        &lt;plugin&gt;
            &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
            &lt;artifactId&gt;spring-boot-maven-plugin&lt;/artifactId&gt;
        &lt;/plugin&gt;
    &lt;/plugins&gt;
&lt;/build&gt;</pre>

### <span style="color: #0b5394;">2.2 Spring Configuration with Java Config</span>

We used @SpringBootApplication instead of the usual @Configuration annotation. This Spring Boot annotation is also annotated with @Configuration. In addition, it sets other configuration like Spring Boot auto configuration:

<pre class="lang:java decode:true ">@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Configuration
@EnableAutoConfiguration
@ComponentScan
public @interface SpringBootApplication {</pre>

&nbsp;

The configuration class does not need to define any bean. All the configuration is automatically set by Spring Boot. Regarding the connection factory, Spring Boot will detect that I included the ActiveMQ dependency on the classpath and will start and configure an embedded broker.

<pre class="lang:java decode:true ">@SpringBootApplication
public class JmsJavaconfigApplication {

    public static void main(String[] args) {
        SpringApplication.run(JmsJavaconfigApplication.class, args);
    }
}</pre>

&nbsp;

If you need to specify a different broker url, you can declare it in the properties. Check <a href="http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-messaging.html#boot-features-activemq" target="_blank" rel="noopener">ActiveMQ support</a> section for further detail.

It is all set now. We will see how to configure a JMS listener in the example in the next section, since it is configured with an annotation.

## <span style="color: #0b5394;">3 A simple example with JMS listener</span>

### <span style="color: #0b5394;">3.1   Sending an order to a JMS queue</span>

The ClientService class is responsible for sending a new order to the JMS queue. In order to accomplish this, it uses a JmsTemplate:

<pre class="lang:java decode:true ">@Service
public class ClientServiceImpl implements ClientService {
    private static final String SIMPLE_QUEUE = "simple.queue";
    
    private final JmsTemplate jmsTemplate;
    
    @Autowired
    public ClientServiceImpl(JmsTemplate jmsTemplate) {
        this.jmsTemplate = jmsTemplate;
    }
    
    @Override
    public void addOrder(Order order) {
        jmsTemplate.convertAndSend(SIMPLE_QUEUE, order);
    }
}</pre>

&nbsp;

Here, we use a JmsTemplate to convert our Order instance and send it to the JMS queue. If you prefer to directly send a message through the _send_ message, you can instead use the new <a href="http://docs.spring.io/autorepo/docs/spring/4.1.6.RELEASE/javadoc-api/org/springframework/jms/core/JmsMessagingTemplate.html" target="_blank" rel="noopener">JmsMessagingTemplate</a>. This is preferable since it uses the more standardized <a href="http://docs.spring.io/autorepo/docs/spring/4.1.6.RELEASE/javadoc-api/org/springframework/messaging/Message.html" target="_blank" rel="noopener">Message</a> class.

### <span style="color: #0b5394;">3.2 Receiving an order sent to the JMS queue</span>

Registering a JMS listener to a JMS listener container is as simple as adding the @JmsListener annotation to the method we want to use. This will create a JMS listener container under the covers that will receive messages sent to the specified queue and delegate them to our listener class:

<pre class="lang:java decode:true ">@Component
public class SimpleListener {
    private final StoreService storeService;
    
    @Autowired
    public SimpleListener(StoreService storeService) {
        this.storeService = storeService;
    }
    
    @JmsListener(destination = "simple.queue")
    public void receiveOrder(Order order) {
        storeService.registerOrder(order);
    }
}</pre>

&nbsp;

The StoreService receives the order and saves it to a list of received orders:

<pre class="lang:java decode:true">@Service
public class StoreServiceImpl implements StoreService {
    private final List&lt;Order&gt; receivedOrders = new ArrayList&lt;&gt;();
    
    @Override
    public void registerOrder(Order order) {
        this.receivedOrders.add(order);
    }
    
    @Override
    public Optional&lt;Order&gt; getReceivedOrder(String id) {
        return receivedOrders.stream().filter(o -&gt; o.getId().equals(id)).findFirst();
    }
}</pre>

### <span style="color: #0b5394;">3.3 Testing the application</span>

Now let’s add a test to check if we did everything correctly:

<pre class="lang:java decode:true">@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = JmsJavaconfigApplication.class)
public class SimpleListenerTest {
    
    @Autowired
    private ClientService clientService;
    
    @Autowired
    private StoreService storeService;
    
    @Test
    public void sendSimpleMessage() {
        clientService.addOrder(new Order("order1"));
        
        Optional&lt;Order&gt; storedOrder = storeService.getReceivedOrder("order1");
        Assert.assertTrue(storedOrder.isPresent());
        Assert.assertEquals("order1", storedOrder.get().getId());
    }
}</pre>

&nbsp;

## <span style="color: #0b5394;">4 Sending a response to another queue with @SendTo</span>

Another addition to Spring JMS is the @SendTo annotation. This annotation allows a listener to send a message to another queue. For example, the following listener receives an order from the “in.queue” and after storing the order, sends a confirmation to the “out.queue”.

<pre class="lang:java decode:true ">@JmsListener(destination = "in.queue")
@SendTo("out.queue")
public String receiveOrder(Order order) {
    storeService.registerOrder(order);
    return order.getId();
}</pre>

&nbsp;

There, we have another listener registered that will process this confirmation id:

<pre class="lang:java decode:true">@JmsListener(destination = "out.queue")
public void receiveOrder(String orderId) {
    registerService.registerOrderId(orderId);
}</pre>

&nbsp;

## <span style="color: #0b5394;">5 Conclusion</span>

With annotation support, it is now much easier to configure a Spring JMS application, taking advantage of asynchronous message retrieval using annotated JMS listeners.

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.

&nbsp;