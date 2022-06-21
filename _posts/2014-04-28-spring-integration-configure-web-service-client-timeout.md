---
id: 25
title: 'Spring Integration &#8211; Configure web service client timeout'
date: 2014-04-28T12:23:00+01:00
author: xpadro
layout: post
permalink: /2014/04/spring-integration-configure-web-service-client-timeout.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2014/04/spring-integration-configure-web.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/6219913651901114507
image: /wp-content/uploads/2014/04/int-timeout-sequence.png
categories:
  - Integration
  - Spring
  - web services
tags:
  - Integration
  - Spring
  - Test
  - web services
---

With the support of Spring Integration, your application can invoke a web service by using an outbound web service gateway. The invocation is handled by this gateway. Hence, you just need to worry about building the request message and handling the response. However, with this approach it is not obvious how to configure additional options like setting timeouts or caching of operations. This article will show how to configure web service client timeout and integrate it with the gateway.

This article is divided in the following sections:

  1. Introduction.
  2. Web service invocation overview.
  3. Configuring a message sender.
  4. The sample application.
  5. Conclusion.

&nbsp;

The source code can be found at <a href="https://github.com/xpadro/spring-integration/tree/master/webservices/ws-timeout" target="_blank" rel="noopener">my Github repository</a>.

## <span style="color: #0b5394;">1 Web service invocation overview</span>

The web service outbound gateway delegates the web service invocation to the <a href="http://projects.spring.io/spring-ws/" target="_blank" rel="noopener">Spring Web Services</a> <a href="http://docs.spring.io/spring-ws/sites/2.0/apidocs/org/springframework/ws/client/core/WebServiceTemplate.html" target="_blank" rel="noopener">WebServiceTemplate</a>. When a message arrives to the outbound gateway, this template uses a message sender in order to create a new connection. The diagram below shows an overview of the flow:

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <img loading="lazy" class="alignnone wp-image-169 size-full" src="http://xpadro.com/wp-content/uploads/2014/04/int-timeout-sequence-1.png" alt="configure web service client timeout diagram" width="886" height="346" srcset="https://xpadro.com/wp-content/uploads/2014/04/int-timeout-sequence-1.png 886w, https://xpadro.com/wp-content/uploads/2014/04/int-timeout-sequence-1-300x117.png 300w, https://xpadro.com/wp-content/uploads/2014/04/int-timeout-sequence-1-768x300.png 768w" sizes="(max-width: 886px) 100vw, 886px" />
</div>

&nbsp;

By default, the web service template sets an <a href="http://docs.spring.io/spring-ws/sites/2.0/apidocs/org/springframework/ws/transport/http/HttpUrlConnectionMessageSender.html" target="_blank" rel="noopener">HttpUrlConnectionMessageSender</a> as its message sender, which is a basic implementation without support for configuration options. This behavior though, can be overridden by setting a more advanced message sender with the capability of setting both read and connection timeouts.

We are going to configure the message sender in the next section.

## <span style="color: #0b5394;">2 Configuring a message sender</span>

We are going to configure a message sender to the outbound gateway. This way, the gateway will set the template’s message sender with the one provided.

The implementation we are providing in the example is the <a href="http://docs.spring.io/spring-ws/sites/2.0/apidocs/org/springframework/ws/transport/http/HttpComponentsMessageSender.html" target="_blank" rel="noopener">HttpComponentsMessageSender</a> class, also from the Spring Web Services project. This message sender allows us to define the following timeouts:

  * **connectionTimeout**: Sets the timeout until the connection is established.
  * **readTimeout**: Sets the socket timeout for the underlying HttpClient. This is the time required for the service to reply.

Configuration:

<pre class="lang:xhtml decode:true ">&lt;bean id="messageSender" class="org.springframework.ws.transport.http.HttpComponentsMessageSender"&gt;
    &lt;property name="connectionTimeout" value="${timeout.connection}"/&gt;
    &lt;property name="readTimeout" value="${timeout.read}"/&gt;
&lt;/bean&gt;</pre>

&nbsp;

The properties file contains the values, which are both set to two seconds:

<span style="font-size: x-small;">timeout.connection=2000</span>  
<span style="font-size: x-small;">timeout.read=2000</span>

Once configured, we add it to the web service outbound gateway configuration:

<pre class="lang:xhtml decode:true ">&lt;int-ws:outbound-gateway uri="http://localhost:8080/spring-ws-courses/courses" 
    marshaller="marshaller" unmarshaller="marshaller" 
    request-channel="requestChannel" message-sender="messageSender"/&gt;</pre>

&nbsp;

To use this message sender, you will need to add the following dependency:

<pre class="lang:xhtml decode:true ">&lt;dependency&gt;
    &lt;groupId&gt;org.apache.httpcomponents&lt;/groupId&gt;
    &lt;artifactId&gt;httpclient&lt;/artifactId&gt;
    &lt;version&gt;4.3.3&lt;/version&gt;
&lt;/dependency&gt;</pre>

&nbsp;

And that’s it; the next section will show the sample application to see how it works.

## <span style="color: #0b5394;">3 The sample application</span>

The flow is simple; it consists in an application that sends a request to a web service and receives a response. The web service source code can be found at <a href="https://github.com/xpadro/spring-samples/tree/master/spring-ws-courses" target="_blank" rel="noopener">github</a>.

<pre class="lang:xhtml decode:true ">&lt;beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:int="http://www.springframework.org/schema/integration"
    xmlns:int-ws="http://www.springframework.org/schema/integration/ws"
    xmlns:oxm="http://www.springframework.org/schema/oxm"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.0.xsd
        http://www.springframework.org/schema/integration http://www.springframework.org/schema/integration/spring-integration.xsd
        http://www.springframework.org/schema/integration/ws http://www.springframework.org/schema/integration/ws/spring-integration-ws.xsd
        http://www.springframework.org/schema/oxm http://www.springframework.org/schema/oxm/spring-oxm-3.0.xsd"&gt;
    
    &lt;context:component-scan base-package="xpadro.spring.integration.ws"/&gt;
    &lt;context:property-placeholder location="classpath:props/service.properties"/&gt;
    
    &lt;!-- System entry --&gt;
    &lt;int:gateway id="systemEntry" default-request-channel="requestChannel" 
        service-interface="xpadro.spring.integration.ws.gateway.CourseService"/&gt;
    
    &lt;!-- Web service invocation --&gt;
    &lt;int-ws:outbound-gateway uri="http://localhost:8080/spring-ws-courses/courses" 
            marshaller="marshaller" unmarshaller="marshaller" 
            request-channel="requestChannel" message-sender="messageSender"/&gt;
    
    &lt;oxm:jaxb2-marshaller id="marshaller" contextPath="xpadro.spring.integration.ws.types" /&gt;
    
    &lt;bean id="messageSender" class="org.springframework.ws.transport.http.HttpComponentsMessageSender"&gt;
        &lt;property name="connectionTimeout" value="${timeout.connection}"/&gt;
        &lt;property name="readTimeout" value="${timeout.read}"/&gt;
    &lt;/bean&gt;

&lt;/beans&gt;</pre>

&nbsp;

The gateway contains the method through which we will enter the messaging system:

<pre class="lang:java decode:true ">public interface CourseService {
    
    @Gateway
    GetCourseResponse getCourse(GetCourseRequest request);
</pre>

&nbsp;

Finally, the test:

<pre class="lang:java decode:true ">@ContextConfiguration(locations = {"/xpadro/spring/integration/ws/config/int-course-config.xml"})
@RunWith(SpringJUnit4ClassRunner.class)
public class TestIntegrationApp {
    
    @Autowired
    private CourseService service;
    
    @Test
    public void invokeNormalOperation() {
        GetCourseRequest request = new GetCourseRequest();
        request.setCourseId("BC-45");
        
        GetCourseResponse response = service.getCourse(request);
        assertNotNull(response);
        assertEquals("Introduction to Java", response.getName());
    }
    
    @Test
    public void invokeTimeoutOperation() {
        try {
            GetCourseRequest request = new GetCourseRequest();
            request.setCourseId("DF-21");
            
            GetCourseResponse response = service.getCourse(request);
            assertNull(response);
        } catch (WebServiceIOException e) {
            assertTrue(e.getCause() instanceof SocketTimeoutException);
        }
    }
}</pre>

&nbsp;

## <span style="color: #0b5394;">4 Conclusion</span>

We have learnt how to set additional options to the web service outbound gateway in order to establish a timeout.

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.