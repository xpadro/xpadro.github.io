---
id: 24
title: 'Spring Integration 4.0: A complete XML-free example'
date: 2014-05-05T09:51:00+01:00
author: xpadro
layout: post
permalink: /2014/05/spring-integration-4-0-a-complete-xml-free-example.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2014/05/spring-integration-40-complete-xml-free.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/6281967422271241394
image: /wp-content/uploads/2014/05/int-v4-full-flow_v2.png
categories:
  - Integration
  - mongoDB
  - Spring
  - Spring WS
tags:
  - Integration
  - Spring
  - web services
---

<div>
  Spring Integration 4.0 is finally <a href="https://spring.io/blog/2014/04/30/spring-integration-4-0-released" target="_blank" rel="noopener">here</a>, and this release comes with very nice features. The one covered in this article is the possibility to configure an integration flow without using XML at all. Those people that don’t like XML will be able to develop an integration application with just using JavaConfig.
</div>

<div>
</div>

<div>
  This article is divided in the following sections:
</div>

<div>
  <ol>
    <li>
      Introduction.
    </li>
    <li>
      An overview of the flow.
    </li>
    <li>
      Spring configuration.
    </li>
    <li>
      Detail of the endpoints.
    </li>
    <li>
      Testing the entire flow.
    </li>
    <li>
      Conclusion.
    </li>
  </ol>
</div>

The source code can be found at <a href="https://github.com/xpadro/spring-integration/tree/master/generic/int-v4-full" target="_blank" rel="noopener">github</a>.

The source code of the web service invoked in this example can be found at the spring-samples repository at <a href="https://github.com/xpadro/spring-samples/tree/master/spring-ws-courses" target="_blank" rel="noopener">github</a>.

## <span style="color: #0b5394;">1 An overview of the flow</span>

The example application shows how to configure several messaging and integration endpoints. The user asks for a course by specifying the course Id. The flow will invoke a web service and return the response to the user. Additionally, some type of courses will be stored to a database.

The flow is as follows:

  * An integration <a href="http://docs.spring.io/spring-integration/docs/4.0.0.RELEASE/reference/html/messaging-endpoints-chapter.html#gateway" target="_blank" rel="noopener">gateway </a>(course service) serves as the entry to the messaging system.
  * A <a href="http://docs.spring.io/spring-integration/docs/4.0.0.RELEASE/reference/html/messaging-transformation-chapter.html#transformer" target="_blank" rel="noopener">transformer </a>builds the request message from the user specified course Id.
  * A web service <a href="http://docs.spring.io/spring-integration/docs/4.0.0.RELEASE/reference/html/ws.html#webservices-outbound" target="_blank" rel="noopener">outbound gateway</a> sends the request to a web service and waits for a response.
  * A <a href="http://docs.spring.io/spring-integration/docs/4.0.0.RELEASE/reference/html/messaging-endpoints-chapter.html#service-activator" target="_blank" rel="noopener">service activator</a> is subscribed to the response channel in order to return the course name to the user.
  * A <a href="http://docs.spring.io/spring-integration/docs/4.0.0.RELEASE/reference/html/messaging-routing-chapter.html#filter" target="_blank" rel="noopener">filter </a>is also subscribed to the response channel. This filter will send some types of courses to a mongodb <a href="http://docs.spring.io/spring-integration/docs/4.0.0.RELEASE/reference/html/mongodb.html#mongodb-outbound-channel-adapter" target="_blank" rel="noopener">channel adapter</a> in order to store the response to a database.

The following diagram better shows how the flow is structured:

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <img loading="lazy" class="alignnone wp-image-174 size-full" src="http://xpadro.com/wp-content/uploads/2014/05/int-v4-full-flow_v2-1.png" alt="Spring Integration 4.0 application flow diagram" width="795" height="533" srcset="https://xpadro.com/wp-content/uploads/2014/05/int-v4-full-flow_v2-1.png 795w, https://xpadro.com/wp-content/uploads/2014/05/int-v4-full-flow_v2-1-300x201.png 300w, https://xpadro.com/wp-content/uploads/2014/05/int-v4-full-flow_v2-1-768x515.png 768w" sizes="(max-width: 795px) 100vw, 795px" />
</div>

&nbsp;

## <span style="color: #0b5394;">2 Spring configuration</span>

As discussed in the introduction section, the entire configuration is defined with JavaConfig. This configuration is split into three files: infrastructure, web service and database configuration. Let’s check it out:

### <span style="color: #0b5394;">2.1 Infrastructure configuration</span>

This configuration file only contains the definition of message channels. The messaging endpoints (transformer, filter, etc&#8230;) are configured with annotations.

**InfrastructureConfiguration.java**

<pre class="lang:java decode:true ">@Configuration
@ComponentScan("xpadro.spring.integration.endpoint")	//@Component
@IntegrationComponentScan("xpadro.spring.integration.gateway")	//@MessagingGateway
@EnableIntegration
@Import({MongoDBConfiguration.class, WebServiceConfiguration.class})
public class InfrastructureConfiguration {
    
    @Bean
    @Description("Entry to the messaging system through the gateway.")
    public MessageChannel requestChannel() {
        return new DirectChannel();
    }
    
    @Bean
    @Description("Sends request messages to the web service outbound gateway")
    public MessageChannel invocationChannel() {
        return new DirectChannel();
    }
    
    @Bean
    @Description("Sends web service responses to both the client and a database")
    public MessageChannel responseChannel() {
        return new PublishSubscribeChannel();
    }
    
    @Bean
    @Description("Stores non filtered messages to the database")
    public MessageChannel storeChannel() {
        return new DirectChannel();
    }
}</pre>

&nbsp;

The @ComponentScan annotation searches for @Component annotated classes, which are our defined messaging endpoints; the filter, the transformer and the service activator.

The @IntegrationComponentScan annotation searches for specific integration annotations. In our example, it will scan the entry gateway which is annotated with @MessagingGateway.

The @EnableIntegration annotation enables integration configuration. For example, method level annotations like @Transformer or @Filter.

### <span style="color: #0b5394;">2.2 Web service configuration</span>

This configuration file configures the web service outbound gateway and its required marshaller.

**WebServiceConfiguration.java**

<pre class="lang:java decode:true ">@Configuration
public class WebServiceConfiguration {
    
    @Bean
    @ServiceActivator(inputChannel = "invocationChannel")
    public MessageHandler wsOutboundGateway() {
        MarshallingWebServiceOutboundGateway gw = new MarshallingWebServiceOutboundGateway("http://localhost:8080/spring-ws-courses/courses", jaxb2Marshaller());
        gw.setOutputChannelName("responseChannel");
        
        return gw;
    }
    
    @Bean
    public Jaxb2Marshaller jaxb2Marshaller() {
        Jaxb2Marshaller marshaller = new Jaxb2Marshaller();
        marshaller.setContextPath("xpadro.spring.integration.ws.types");
        
        return marshaller;
    }
}</pre>

&nbsp;

The gateway allows us to define its output channel but not the input channel. We need to annotate the adapter with @ServiceActivator in order to subscribe it to the invocation channel and avoid having to autowire it in the message channel bean definition.

### <span style="color: #0b5394;">2.3 Database configuration</span>

This configuration file defines all necessary beans to set up <a href="https://www.mongodb.org/" target="_blank" rel="noopener">mongoDB</a>. It also defines the mongoDB outbound channel adapter.

**MongoDBConfiguration.java**

<pre class="lang:java decode:true ">@Configuration
public class MongoDBConfiguration {
    
    @Bean
    public MongoDbFactory mongoDbFactory() throws Exception {
        return new SimpleMongoDbFactory(new MongoClient(), "si4Db");
    }
    
    @Bean
    @ServiceActivator(inputChannel = "storeChannel")
    public MessageHandler mongodbAdapter() throws Exception {
        MongoDbStoringMessageHandler adapter = new MongoDbStoringMessageHandler(mongoDbFactory());
        adapter.setCollectionNameExpression(new LiteralExpression("courses"));
        
        return adapter;
    }
}</pre>

&nbsp;

Like the web service gateway, we can’t set the input channel to the adapter. I also have done that by specifying the input channel in the @ServiceActivator annotation.

## <span style="color: #0b5394;">3 Detail of the endpoints</span>

The first endpoint of the flow is the integration gateway, which will put the argument (courseId) into the payload of a message and send it to the request channel.

<pre class="lang:java decode:true ">@MessagingGateway(name = "entryGateway", defaultRequestChannel = "requestChannel")
public interface CourseService {
    
    public String findCourse(String courseId);
}</pre>

&nbsp;

The message containing the course id will reach the transformer. This endpoint will build the request object that the web service is expecting:

<pre class="lang:java decode:true ">@Component
public class CourseRequestBuilder {
    private Logger logger = LoggerFactory.getLogger(this.getClass());
    
    @Transformer(inputChannel="requestChannel", outputChannel="invocationChannel")
    public GetCourseRequest buildRequest(Message&lt;String&gt; msg) {
        logger.info("Building request for course [{}]", msg.getPayload());
        GetCourseRequest request = new GetCourseRequest();
        request.setCourseId(msg.getPayload());
        
        return request;
    }
}</pre>

&nbsp;

Subscribed to the response channel, which is the channel where the web service reply will be sent, there’s a service activator that will receive the response message and deliver the course name to the client:

<pre class="lang:java decode:true ">@Component
public class CourseResponseHandler {
    private Logger logger = LoggerFactory.getLogger(this.getClass());
    
    @ServiceActivator(inputChannel="responseChannel")
    public String getResponse(Message&lt;GetCourseResponse&gt; msg) {
        GetCourseResponse course = msg.getPayload();
        logger.info("Course with ID [{}] received: {}", course.getCourseId(), course.getName());
        
        return course.getName();
    }
}</pre>

&nbsp;

Also subscribed to the response channel, a filter will decide based on its type, if the course is required to be stored to a database:

<pre class="lang:java decode:true ">@Component
public class StoredCoursesFilter {
    private Logger logger = LoggerFactory.getLogger(this.getClass());
    
    @Filter(inputChannel="responseChannel", outputChannel="storeChannel")
    public boolean filterCourse(Message&lt;GetCourseResponse&gt; msg) {
        if (!msg.getPayload().getCourseId().startsWith("BC-")) {
            logger.info("Course [{}] filtered. Not a BF course", msg.getPayload().getCourseId());
            return false;
        }
        
        logger.info("Course [{}] validated. Storing to database", msg.getPayload().getCourseId());
        return true;
    }
}</pre>

&nbsp;

## <span style="color: #0b5394;">4 Testing the entire flow</span>

The following client will send two requests; a BC type course request that will be stored to the database and a DF type course that will be finally filtered:

<pre class="lang:java decode:true ">@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes={InfrastructureConfiguration.class})
public class TestApp {
    @Autowired
    CourseService service;
    
    @Test
    public void testFlow() {
        String courseName = service.findCourse("BC-45");
        assertNotNull(courseName);
        assertEquals("Introduction to Java", courseName);
        
        courseName = service.findCourse("DF-21");
        assertNotNull(courseName);
        assertEquals("Functional Programming Principles in Scala", courseName);
	}
}</pre>

&nbsp;

This will result in the following console output:

<div>
  <span style="font-size: x-small;">CourseRequestBuilder|Building request for course [BC-45]</span>
</div>

<div>
  <span style="font-size: x-small;">CourseResponseHandler|Course with ID [BC-45] received: Introduction to Java</span>
</div>

<div>
  <span style="font-size: x-small;">StoredCoursesFilter|Course [BC-45] validated. Storing to database</span>
</div>

<div>
  <span style="font-size: x-small;">CourseRequestBuilder|Building request for course [DF-21]</span>
</div>

<div>
  <span style="font-size: x-small;">CourseResponseHandler|Course with ID [DF-21] received: Functional Programming Principles in Scala</span>
</div>

<div>
  <span style="font-size: x-small;">StoredCoursesFilter|Course [DF-21] filtered. Not a BF course</span></p>
</div>

## <span style="color: #0b5394;">5 Conclusion</span>

We have learnt how to set up and test an application powered with Spring Integration using no XML configuration. Stay tuned, because Spring Integration Java DSL with Spring Integration <a href="https://github.com/spring-projects/spring-integration-extensions/tree/master/spring-integration-java-dsl" target="_blank" rel="noopener">extensions </a>is on its way!

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.

&nbsp;