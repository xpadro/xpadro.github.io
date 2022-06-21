---
id: 29
title: Retry web service operations with RequestHandlerRetryAdvice
date: 2013-12-31T15:31:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/12/31/retry-web-service-operations-with-requesthandlerretryadvice/
permalink: /2013/12/retry-web-service-operations-with-requesthandlerretryadvice.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/12/retry-web-service-operations-with.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/3613601074429811636
categories:
  - Integration
  - mongoDB
  - Spring
tags:
  - Integration
  - MongoDB
  - Spring
  - web services
---

Sometimes when invoking a web service, we may be interested in retrying the operation in case an error occurs. When using Spring Integration, we can achieve this functionality with <a href="http://docs.spring.io/spring-integration/docs/3.0.0.RELEASE/api/org/springframework/integration/handler/advice/RequestHandlerRetryAdvice.html" target="_blank" rel="noopener">RequestHandlerRetryAdvice</a> class. This class will allow us to retry the operation for a specified number of times before giving up and throwing an exception. This post will show you how to accomplish this.

The test application will invoke a web service and if it fails to respond, it will wait for a specified time and try it again until it receives a response or it reaches a retry limit. If the limit is reached, the failed request will be stored into a database. Mainly, this post shows an example of the following:

  * Using an <a href="http://docs.spring.io/spring-integration/docs/3.0.0.RELEASE/reference/html/ws.html#webservices-outbound" target="_blank" rel="noopener">outbound gateway</a> to invoke a web service
  * Configuring a <a href="https://spring.io/blog/2012/10/09/what-s-new-in-spring-integration-2-2-part-4-retry-and-more" target="_blank" rel="noopener">retry advice</a> to retry the operation if necessary
  * <a href="http://docs.spring.io/spring-integration/docs/3.0.0.RELEASE/reference/html/mongodb.html" target="_blank" rel="noopener">MongoDB</a> integration.

The source code of the application can be found at <a href="https://github.com/xpadro/spring-integration/tree/master/webservices/ws-retry-adv" target="_blank" rel="noopener">my Github repository</a>.  
You can also get the source code of the web service project that is called by the application at <a href="https://github.com/xpadro/spring-integration/tree/master/webservices/spring-ws" target="_blank" rel="noopener">my Github repository</a>.

## <span style="color: #0b5394;">1 Web service invocation</span>

**Use case**: The client invokes the web service and receives a response.

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://1.bp.blogspot.com/-82ZZ11JUJ2k/VUZLPdh2utI/AAAAAAAAEns/JrMp59G-Q1o/s1600/grafic1.png"><img loading="lazy" class="alignnone" src="https://1.bp.blogspot.com/-82ZZ11JUJ2k/VUZLPdh2utI/AAAAAAAAEns/JrMp59G-Q1o/s1600/grafic1.png" alt="web service invocation diagram" width="640" height="330" border="0" /></a>
</div>

The request enters the messaging system through the &#8220;system entry&#8221; gateway. It then reaches the outbound gateway, invokes the web service and waits for the response. Once received, the response is sent to the response channel.

The above image is the result of this configuration:

<pre class="lang:xhtml decode:true ">&lt;context:component-scan base-package="xpadro.spring.integration" /&gt;

&lt;!-- Initial service request --&gt;
&lt;int:gateway id="systemEntry" default-request-channel="requestChannel"
    service-interface="xpadro.spring.integration.gateway.ClientService" /&gt;
    
&lt;int:channel id="requestChannel"&gt;
    &lt;int:queue /&gt;
&lt;/int:channel&gt;

&lt;int-ws:outbound-gateway id="marshallingGateway"
    request-channel="requestChannel" reply-channel="responseChannel"
    uri="http://localhost:8080/spring-ws/orders" marshaller="marshaller"
    unmarshaller="marshaller"&gt;
    
    &lt;int:poller fixed-rate="500" /&gt;
&lt;/int-ws:outbound-gateway&gt;

&lt;oxm:jaxb2-marshaller id="marshaller" contextPath="xpadro.spring.integration.types" /&gt;


&lt;!-- Service is running - Response received --&gt;
&lt;int:channel id="responseChannel" /&gt;

&lt;int:service-activator ref="clientServiceActivator" method="handleServiceResult" input-channel="responseChannel" /&gt;</pre>

&nbsp;

Mapped to the response channel there&#8217;s a service activator which just logs the result.

<u>TestInvocation.java: Sends the request to the entry gateway</u>

<pre class="lang:java decode:true ">@ContextConfiguration({"classpath:xpadro/spring/integration/config/int-config.xml",
    "classpath:xpadro/spring/integration/config/mongodb-config.xml"})
@RunWith(SpringJUnit4ClassRunner.class)
public class TestInvocation {
    private Logger logger = LoggerFactory.getLogger(this.getClass());
    
    @Autowired
    private ClientService service;
    
    @Test
    public void testInvocation() throws InterruptedException, ExecutionException {
        logger.info("Initiating service request...");
        
        ClientDataRequest request = new ClientDataRequest();
        request.setClientId("123");
        request.setProductId("XA-55");
        request.setQuantity(new BigInteger("5"));

        service.invoke(request);
        
        logger.info("Doing other stuff...");
        Thread.sleep(60000);
    }
}</pre>

&nbsp;

With this configuration, if the service invocation fails, a <a href="http://docs.spring.io/spring-integration/api/org/springframework/integration/MessagingException.html" target="_blank" rel="noopener">MessagingException</a> will be raised and sent to the error channel. In the next section, we will add the retry configuration.

## <span style="color: #0b5394;">2 Adding the retry advice</span>

**Use case**: The initial request failed because the service is not active. We will retry the operation until a response is received from the service.

In this case, we need to add the retry advice to the web service outbound gateway:

<pre class="lang:xhtml decode:true ">&lt;int-ws:outbound-gateway id="marshallingGateway" interceptor="clientInterceptor"
    request-channel="requestChannel" reply-channel="responseChannel"
    uri="http://localhost:8080/spring-ws/orders" marshaller="marshaller"
    unmarshaller="marshaller"&gt;
    
    &lt;int:poller fixed-rate="500" /&gt;
    
    &lt;int-ws:request-handler-advice-chain&gt;
        &lt;ref bean="retryAdvice" /&gt;
    &lt;/int-ws:request-handler-advice-chain&gt;
&lt;/int-ws:outbound-gateway&gt;</pre>

&nbsp;

Now the web service outbound gateway will delegate the invocation to the retry advice, which will try the operation as many times as specified until it gets a response from the service.

Let&#8217;s define the retry advice:

<pre class="lang:xhtml decode:true ">&lt;bean id="retryAdvice" class="org.springframework.integration.handler.advice.RequestHandlerRetryAdvice" &gt;
    &lt;property name="retryTemplate"&gt;
        &lt;bean class="org.springframework.retry.support.RetryTemplate"&gt;
            &lt;property name="backOffPolicy"&gt;
                &lt;bean class="org.springframework.retry.backoff.FixedBackOffPolicy"&gt;
                    &lt;property name="backOffPeriod" value="4000" /&gt;
                &lt;/bean&gt;
            &lt;/property&gt;
            &lt;property name="retryPolicy"&gt;
                &lt;bean class="org.springframework.retry.policy.SimpleRetryPolicy"&gt;
                    &lt;property name="maxAttempts" value="4" /&gt;
                &lt;/bean&gt;
            &lt;/property&gt;
        &lt;/bean&gt;
    &lt;/property&gt;
&lt;/bean&gt;</pre>

&nbsp;

To accomplish its objective, the advice uses a RetryTemplate, which is provided by the <a href="https://github.com/spring-projects/spring-retry" target="_blank" rel="noopener">Spring Retry</a> project. We can customize its behavior by defining _backoff_ and _retry_ policies.

**Backoff policy**: Establishes a period of time between each retry. The more interesting types are:

  * _FixedBackOffPolicy_: Used in our example. It will wait for the same specified amount of time between each retry.
  * _ExponentialBackOffPolicy_: Starting with a determined amount of time, it will double the time on each retry. You can change the default behavior by establishing a multiplier.

**Retry policy**: Establishes how many times will retry the failed operation. Some of the types:

  * _SimpleRetryPolicy_: Used in our example. Specifies a retry attempts limit.
  * _ExceptionClassifierRetryPolicy_: Allows us to establish a different maxAttempts depending on the exception raised.
  * _TimeoutRetryPolicy_: Will keep retrying until a timeout is reached.

&nbsp;

## <span style="color: #0b5394;">3 No luck, logging the failed request</span>

**Use case**: The service won&#8217;t recover, storing the request to the database.

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://1.bp.blogspot.com/-ZNorYO_qg6A/VUZYodSjSNI/AAAAAAAAEok/1EQXRluB0Gs/s1600/pic2..png"><img loading="lazy" class="alignnone" src="https://1.bp.blogspot.com/-ZNorYO_qg6A/VUZYodSjSNI/AAAAAAAAEok/1EQXRluB0Gs/s1600/pic2..png" alt="log to database flow diagram" width="640" height="378" border="0" /></a>
</div>

The final part of the configuration is the following:

<pre class="lang:xhtml decode:true ">&lt;!-- Log failed invocation --&gt;
&lt;int:service-activator ref="clientServiceActivator" method="handleFailedInvocation" input-channel="errorChannel" output-channel="logChannel" /&gt;

&lt;int:channel id="logChannel" /&gt;

&lt;bean id="mongoDbFactory" class="org.springframework.data.mongodb.core.SimpleMongoDbFactory"&gt;
    &lt;constructor-arg&gt;
        &lt;bean class="com.mongodb.Mongo"/&gt;
    &lt;/constructor-arg&gt;
    &lt;constructor-arg value="test"/&gt;
&lt;/bean&gt;

&lt;int-mongodb:outbound-channel-adapter id="mongodbAdapter" channel="logChannel"
    collection-name="failedRequests" mongodb-factory="mongoDbFactory" /&gt;</pre>

The service activator subscribed to the error channel will retrieve the failed message and send it to the outbound adapter, which will insert it to a mongoDB database.

The service activator:

<pre class="lang:java decode:true ">public Message&lt;?&gt; handleFailedInvocation(MessagingException exception) {
    logger.info("Failed to succesfully invoke service. Logging to DB...");
    return exception.getFailedMessage();
}</pre>

&nbsp;

If we not succeed in obtaining a response from the service, the request will be stored into the database:

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://2.bp.blogspot.com/-paLyHdiuVC8/VUZLPUSlI3I/AAAAAAAAEno/yuAHTb17IwQ/s1600/mongodb.png"><img loading="lazy" class="alignnone" src="https://2.bp.blogspot.com/-paLyHdiuVC8/VUZLPUSlI3I/AAAAAAAAEno/yuAHTb17IwQ/s1600/mongodb.png" alt="request stored in mongoDb" width="640" height="126" border="0" /></a>
</div>

<div>
</div>

## <span style="color: #0b5394;">4 Conclusion</span>

We&#8217;ve learnt how Spring Integration gets support from the Spring Retry project in order to achieve retry of operations. We&#8217;ve used the _int-ws:request-handler-advice-chain_, but the &#8216;int&#8217; namespace also supports this element to add this functionality to other types of endpoints.

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.