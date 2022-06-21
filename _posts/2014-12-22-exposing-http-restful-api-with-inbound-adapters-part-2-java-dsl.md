---
id: 17
title: Exposing HTTP Restful API with Inbound Adapters. Part 2 (Java DSL)
date: 2014-12-22T07:29:00+01:00
author: xpadro
layout: post
permalink: /2014/12/exposing-http-restful-api-with-inbound-adapters-part-2-java-dsl.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2014/12/exposing-http-restful-api-with-inbound_22.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/8256222317059075202
categories:
  - Integration
  - Spring
tags:
  - Integration
  - REST
  - Spring
  - Test
  - web services
---

In the [previous part](http://xpadro.com/2014/12/exposing-http-restful-api-with-inbound.html) of this tutorial, we implemented an application exposing a Restful API using XML configuration. This part will re-implement this application using [Spring Integration Java DSL](https://github.com/spring-projects/spring-integration-java-dsl/wiki/Spring-Integration-Java-DSL-Reference).

The application is implemented with Java 8, but when Java 8 specific code is used (for example, when using lambdas), I will also show you how to do it in Java 7. Anyway, I shared both versions at Github in case you want to check it out:

[Java 7 Java DSL example](https://github.com/xpadro/spring-integration/tree/master/http/http-dsl7)

[Java 8 Java DSL example](https://github.com/xpadro/spring-integration/tree/master/http/http-dsl)

This post is divided into the following sections

  1. Introduction
  2. Application configuration
  3. Get operation
  4. Put and post operations
  5. Delete operation
  6. Conclusion

&nbsp;

## <span style="color: #0b5394;">1 Application configuration</span>

In the [web.xml](https://github.com/xpadro/spring-integration/blob/master/int-http-dsl/src/main/webapp/WEB-INF/web.xml) file, the dispatcher servlet is configured to use Java Config:

<pre class="lang:xhtml decode:true ">&lt;servlet&gt;
    &lt;servlet-name&gt;springServlet&lt;/servlet-name&gt;
    &lt;servlet-class&gt;org.springframework.web.servlet.DispatcherServlet&lt;/servlet-class&gt;
    &lt;init-param&gt;
        &lt;param-name&gt;contextClass&lt;/param-name&gt;
        &lt;param-value&gt;org.springframework.web.context.support.AnnotationConfigWebApplicationContext&lt;/param-value&gt;
    &lt;/init-param&gt;
    &lt;init-param&gt;
        &lt;param-name&gt;contextConfigLocation&lt;/param-name&gt;
        &lt;param-value&gt;xpadro.spring.integration.server.configuration&lt;/param-value&gt;
    &lt;/init-param&gt;
&lt;/servlet&gt;
&lt;servlet-mapping&gt;
    &lt;servlet-name&gt;springServlet&lt;/servlet-name&gt;
    &lt;url-pattern&gt;/spring/*&lt;/url-pattern&gt;
&lt;/servlet-mapping&gt;</pre>

&nbsp;

In the [pom.xml](https://github.com/xpadro/spring-integration/blob/master/int-http-dsl/pom.xml) file, we include the Spring Integration Java DSL dependency:

<pre class="lang:xhtml decode:true ">&lt;properties&gt;
    &lt;spring-version&gt;4.1.3.RELEASE&lt;/spring-version&gt;
    &lt;spring-integration-version&gt;4.1.0.RELEASE&lt;/spring-integration-version&gt;
    &lt;slf4j-version&gt;1.7.5&lt;/slf4j-version&gt;
    &lt;junit-version&gt;4.9&lt;/junit-version&gt;
    &lt;jackson-version&gt;2.3.0&lt;/jackson-version&gt;
&lt;/properties&gt;

&lt;dependencies&gt;
    &lt;!-- Spring Framework - Core --&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework&lt;/groupId&gt;
        &lt;artifactId&gt;spring-context&lt;/artifactId&gt;
        &lt;version&gt;${spring-version}&lt;/version&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework&lt;/groupId&gt;
        &lt;artifactId&gt;spring-webmvc&lt;/artifactId&gt;
        &lt;version&gt;${spring-version}&lt;/version&gt;
    &lt;/dependency&gt;
    
    &lt;!-- Spring Framework - Integration --&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework.integration&lt;/groupId&gt;
        &lt;artifactId&gt;spring-integration-core&lt;/artifactId&gt;
        &lt;version&gt;${spring-integration-version}&lt;/version&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework.integration&lt;/groupId&gt;
        &lt;artifactId&gt;spring-integration-http&lt;/artifactId&gt;
        &lt;version&gt;${spring-integration-version}&lt;/version&gt;
    &lt;/dependency&gt;
    
    &lt;!-- Spring Integration - Java DSL --&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework.integration&lt;/groupId&gt;
        &lt;artifactId&gt;spring-integration-java-dsl&lt;/artifactId&gt;
        &lt;version&gt;1.0.0.RELEASE&lt;/version&gt;
    &lt;/dependency&gt;
    
    &lt;!-- JSON --&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;com.fasterxml.jackson.core&lt;/groupId&gt;
        &lt;artifactId&gt;jackson-core&lt;/artifactId&gt;
        &lt;version&gt;${jackson-version}&lt;/version&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;com.fasterxml.jackson.core&lt;/groupId&gt;
        &lt;artifactId&gt;jackson-databind&lt;/artifactId&gt;
        &lt;version&gt;${jackson-version}&lt;/version&gt;
    &lt;/dependency&gt;
    
    &lt;!-- Testing --&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework&lt;/groupId&gt;
        &lt;artifactId&gt;spring-test&lt;/artifactId&gt;
        &lt;version&gt;${spring-version}&lt;/version&gt;
        &lt;scope&gt;test&lt;/scope&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;junit&lt;/groupId&gt;
        &lt;artifactId&gt;junit&lt;/artifactId&gt;
        &lt;version&gt;${junit-version}&lt;/version&gt;
        &lt;scope&gt;test&lt;/scope&gt;
    &lt;/dependency&gt;
    
    &lt;!-- Logging --&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.slf4j&lt;/groupId&gt;
        &lt;artifactId&gt;slf4j-api&lt;/artifactId&gt;
        &lt;version&gt;${slf4j-version}&lt;/version&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.slf4j&lt;/groupId&gt;
        &lt;artifactId&gt;slf4j-log4j12&lt;/artifactId&gt;
        &lt;version&gt;${slf4j-version}&lt;/version&gt;
    &lt;/dependency&gt;
&lt;/dependencies&gt;</pre>

&nbsp;

<u>InfrastructureConfiguration.java</u>

The configuration class contains bean and flow definitions.

<pre class="lang:java decode:true ">@Configuration
@ComponentScan("xpadro.spring.integration.server")
@EnableIntegration
public class InfrastructureConfiguration {
    
    @Bean
    public ExpressionParser parser() {
        return new SpelExpressionParser();
    }
    
    @Bean
    public HeaderMapper&lt;HttpHeaders&gt; headerMapper() {
        return new DefaultHttpHeaderMapper();
    }
    
    //flow and endpoint definitions
}</pre>

&nbsp;

In order to parse payload expressions, we define a bean parser, using an [SpELExpressionParser](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/expression/spel/standard/SpelExpressionParser.html).

The header mapper will later be registered as a property of inbound gateways, in order to map HTTP headers from/to message headers.

The detail of the flows and endpoints defined in this configuration class is explained in each of the following sections.

## <span style="color: #0b5394;">2 Get operation</span>

Our first step is to define the HTTP inbound gateway that will handle GET requests.

<pre class="lang:java decode:true ">@Bean
public MessagingGatewaySupport httpGetGate() {
    HttpRequestHandlingMessagingGateway handler = new HttpRequestHandlingMessagingGateway();
    handler.setRequestMapping(createMapping(new HttpMethod[]{HttpMethod.GET}, "/persons/{personId}"));
    handler.setPayloadExpression(parser().parseExpression("#pathVariables.personId"));
    handler.setHeaderMapper(headerMapper());
    
    return handler;
}

private RequestMapping createMapping(HttpMethod[] method, String... path) {
    RequestMapping requestMapping = new RequestMapping();
    requestMapping.setMethods(method);
    requestMapping.setConsumes("application/json");
    requestMapping.setProduces("application/json");
    requestMapping.setPathPatterns(path);
    
    return requestMapping;
}</pre>

&nbsp;

The createMapping method is the Java alternative to the request-mapping XML element seen in the previous part of the tutorial. In this case, we can also use it to define the request path and supported methods.

Now that we have our gateway set, let’s define the flow that will serve GET requests (remember you can check a diagram of the full flow in the [previous part](http://xpadro.blogspot.com/2014/12/exposing-http-restful-api-with-inbound.html) of the tutorial):

<pre class="lang:java decode:true ">@Bean
public IntegrationFlow httpGetFlow() {
    return IntegrationFlows.from(httpGetGate()).channel("httpGetChannel").handle("personEndpoint", "get").get();
}</pre>

&nbsp;

The flow works as follows:

  * **from(httpGetGate())**: Get messages received by the HTTP Inbound Gateway.
  * **channel(“httpGetChannel”)**: Register a new [DirectChannel](http://docs.spring.io/spring-integration/reference/html/messaging-channels-section.html) bean and send the message received to it.
  * **handle(“personEndpoint”, “get”)**: Messages sent to the previous channel will be consumed by our personEndpoint bean, invoking its get method.

Since we are using a gateway, the response of the personEndpoint will be sent back to the client.

I am showing the personEndpoint for convenience, since it’s actually the same as in the XML application:

<pre class="lang:java decode:true ">@Component
public class PersonEndpoint {
    private static final String STATUSCODE_HEADER = "http_statusCode";
    
    @Autowired
    private PersonService service;
    
    public Message&lt;?&gt; get(Message&lt;String&gt; msg) {
        long id = Long.valueOf(msg.getPayload());
        ServerPerson person = service.getPerson(id);
        
        if (person == null) {
            return MessageBuilder.fromMessage(msg)
                .copyHeadersIfAbsent(msg.getHeaders())
                .setHeader(STATUSCODE_HEADER, HttpStatus.NOT_FOUND)
                .build(); 
        }
        
        return MessageBuilder.withPayload(person)
            .copyHeadersIfAbsent(msg.getHeaders())
            .setHeader(STATUSCODE_HEADER, HttpStatus.OK)
            .build();
    }
    
    //other operations
}</pre>

&nbsp;

GetOperationsTest uses a [RestTemplate](http://spring.io/guides/gs/consuming-rest/) to test the exposed HTTP GET integration flow:

<pre class="lang:java decode:true ">@RunWith(BlockJUnit4ClassRunner.class)
public class GetOperationsTest {
    private static final String URL = "http://localhost:8081/int-http-dsl/spring/persons/{personId}";
    private final RestTemplate restTemplate = new RestTemplate();
    
    private HttpHeaders buildHeaders() {
        HttpHeaders headers = new HttpHeaders();
        headers.setAccept(Arrays.asList(MediaType.APPLICATION_JSON));
        headers.setContentType(MediaType.APPLICATION_JSON); 
        
        return headers;
    }
    
    @Test
    public void getResource_responseIsConvertedToPerson() {
        HttpEntity&lt;Integer&gt; entity = new HttpEntity&lt;&gt;(buildHeaders());
        ResponseEntity&lt;ClientPerson&gt; response = restTemplate.exchange(URL, HttpMethod.GET, entity, ClientPerson.class, 1);
        assertEquals("John" , response.getBody().getName());
        assertEquals(HttpStatus.OK, response.getStatusCode());
    }
    
    //more tests
}</pre>

&nbsp;

I won’t show the full class since it is the same as in the XML example.

## <span style="color: #0b5394;">3 Put and post operations</span>

Continuing with our Restful API application example, we define a bean for the HTTP inbound channel adapter.  You may notice that we are creating a new Gateway. The reason is that inbound channel adapter is internally implemented as a gateway that is not expecting a reply.

<pre class="lang:java decode:true ">@Bean
public MessagingGatewaySupport httpPostPutGate() {
    HttpRequestHandlingMessagingGateway handler = new HttpRequestHandlingMessagingGateway();
    handler.setRequestMapping(createMapping(new HttpMethod[]{HttpMethod.PUT, HttpMethod.POST}, "/persons", "/persons/{personId}"));
    handler.setStatusCodeExpression(parser().parseExpression("T(org.springframework.http.HttpStatus).NO_CONTENT"));
    handler.setRequestPayloadType(ServerPerson.class);
    handler.setHeaderMapper(headerMapper());
    
    return handler;
}</pre>

&nbsp;

We are again using the parser to resolve the returned status code expression.

The former XML attribute request-payload-type of the inbound adapter is now set as a property of the gateway.

The flow that handles both PUT and POST operations uses a [router](https://github.com/spring-projects/spring-integration-java-dsl/wiki/Spring-Integration-Java-DSL-Reference#routers) to send the message to the appropriate endpoint, depending on the HTTP method received:

<pre class="lang:java decode:true ">@Bean
public IntegrationFlow httpPostPutFlow() {
    return IntegrationFlows.from(httpPostPutGate()).channel("routeRequest").route("headers.http_requestMethod", 
        m -&gt; m.prefix("http").suffix("Channel")
            .channelMapping("PUT", "Put")
            .channelMapping("POST", "Post")
    ).get();
}

@Bean
public IntegrationFlow httpPostFlow() {
    return IntegrationFlows.from("httpPostChannel").handle("personEndpoint", "post").get();
}

@Bean
public IntegrationFlow httpPutFlow() {
    return IntegrationFlows.from("httpPutChannel").handle("personEndpoint", "put").get();
}</pre>

&nbsp;

The flow is executed the following way:

  * **from(httpPostPutGate())**:Get messages received by the HTTP Inbound adapter.
  * **channel(“routeRequest”)**: Register a DirectChannel bean and send the message received to it.
  * **route(&#8230;)**: Messages sent to the previous channel will be handled by a router, which will redirect them based on the HTTP method received (http_requestMethod header). The destination channel is resolved applying the prefix and suffix. For example, if the HTTP method is PUT, the resolved channel will be httpPutChannel, which is a bean also defined in this configuration class.

Subflows (httpPutFlow and httpPostFlow) will receive messages from the router and handle them in our personEndpoint.

<pre class="lang:java decode:true ">@Component
public class PersonEndpoint {
    @Autowired
    private PersonService service;
    
    //Get operation
    
    public void put(Message&lt;ServerPerson&gt; msg) {
        service.updatePerson(msg.getPayload());
    }
    
    public void post(Message&lt;ServerPerson&gt; msg) {
        service.insertPerson(msg.getPayload());
    }
}</pre>

&nbsp;

Since we defined an inbound adapter, no response from the endpoint is expected.

In the router definition we used Java 8 lambdas. I told you I would show the alternative in Java 7, so a promise is a promise:

<pre class="lang:java decode:true ">@Bean
public IntegrationFlow httpPostPutFlow() {
    return IntegrationFlows.from(httpPostPutGate()).channel("routeRequest").route("headers.http_requestMethod", 
        new Consumer&lt;RouterSpec&lt;ExpressionEvaluatingRouter&gt;&gt;() {
            @Override
            public void accept(RouterSpec&lt;ExpressionEvaluatingRouter&gt; spec) {
                spec.prefix("http").suffix("Channel")
                    .channelMapping("PUT", "Put")
                    .channelMapping("POST", "Post");
            }
        }
    ).get();
}</pre>

&nbsp;

A little bit longer, isn’t it?

The PUT flow is tested by the PutOperationsTest class:

<pre class="lang:java decode:true ">@RunWith(BlockJUnit4ClassRunner.class)
public class PutOperationsTest {
    private static final String URL = "http://localhost:8081/int-http-dsl/spring/persons/{personId}";
    private final RestTemplate restTemplate = new RestTemplate();
    
    //build headers method
    
    @Test
    public void updateResource_noContentStatusCodeReturned() {
        HttpEntity&lt;Integer&gt; getEntity = new HttpEntity&lt;&gt;(buildHeaders());
        ResponseEntity&lt;ClientPerson&gt; response = restTemplate.exchange(URL, HttpMethod.GET, getEntity, ClientPerson.class, 4);
        ClientPerson person = response.getBody();
        person.setName("Sandra");
        HttpEntity&lt;ClientPerson&gt; putEntity = new HttpEntity&lt;ClientPerson&gt;(person, buildHeaders());
        
        response = restTemplate.exchange(URL, HttpMethod.PUT, putEntity, ClientPerson.class, 4);
        assertEquals(HttpStatus.NO_CONTENT, response.getStatusCode());
        
        response = restTemplate.exchange(URL, HttpMethod.GET, getEntity, ClientPerson.class, 4);
        person = response.getBody();
        assertEquals("Sandra", person.getName());
    }
}</pre>

&nbsp;

The POST flow is tested by the PostOperationsTest class:

<pre class="lang:java decode:true">@RunWith(BlockJUnit4ClassRunner.class)
public class PostOperationsTest {
    private static final String POST_URL = "http://localhost:8081/int-http-dsl/spring/persons";
    private static final String GET_URL = "http://localhost:8081/int-http-dsl/spring/persons/{personId}";
    private final RestTemplate restTemplate = new RestTemplate();
    
    //build headers method
    
    @Test
    public void addResource_noContentStatusCodeReturned() {
        ClientPerson person = new ClientPerson(9, "Jana");
        HttpEntity&lt;ClientPerson&gt; entity = new HttpEntity&lt;ClientPerson&gt;(person, buildHeaders());
        
        ResponseEntity&lt;ClientPerson&gt; response = restTemplate.exchange(POST_URL, HttpMethod.POST, entity, ClientPerson.class);
        assertEquals(HttpStatus.NO_CONTENT, response.getStatusCode());
        
        HttpEntity&lt;Integer&gt; getEntity = new HttpEntity&lt;&gt;(buildHeaders());
        response = restTemplate.exchange(GET_URL, HttpMethod.GET, getEntity, ClientPerson.class, 9);
        person = response.getBody();
        assertEquals("Jana", person.getName());
    }
}</pre>

&nbsp;

## <span style="color: #0b5394;">4 Delete operation</span>

With this operation we complete our application. The entry point is defined by the following bean:

<pre class="lang:java decode:true ">@Bean
public MessagingGatewaySupport httpDeleteGate() {
    HttpRequestHandlingMessagingGateway handler = new HttpRequestHandlingMessagingGateway();
    handler.setRequestMapping(createMapping(new HttpMethod[]{HttpMethod.DELETE}, "/persons/{personId}"));
    handler.setStatusCodeExpression(parser().parseExpression("T(org.springframework.http.HttpStatus).NO_CONTENT"));
    handler.setPayloadExpression(parser().parseExpression("#pathVariables.personId"));
    handler.setHeaderMapper(headerMapper());
    
    return handler;
}</pre>

The configuration is pretty similar to the PutPost gateway. I won’t explain it again.

The delete flow sends the deletion request to the personEndpoint:

<pre class="lang:java decode:true ">@Bean
public IntegrationFlow httpDeleteFlow() {
    return IntegrationFlows.from(httpDeleteGate()).channel("httpDeleteChannel").handle("personEndpoint", "delete").get();
}</pre>

&nbsp;

And our bean will request the service to delete the resource:

<pre class="lang:java decode:true ">public void delete(Message&lt;String&gt; msg) {
    long id = Long.valueOf(msg.getPayload());
    service.deletePerson(id);
}</pre>

&nbsp;

The test asserts that the resource no longer exists after deletion:

<pre class="lang:java decode:true">@RunWith(BlockJUnit4ClassRunner.class)
public class DeleteOperationsTest {
    private static final String URL = "http://localhost:8081/int-http-dsl/spring/persons/{personId}";
    private final RestTemplate restTemplate = new RestTemplate();
    
    //build headers method
    
    @Test
    public void deleteResource_noContentStatusCodeReturned() {
        HttpEntity&lt;Integer&gt; entity = new HttpEntity&lt;&gt;(buildHeaders());
        ResponseEntity&lt;ClientPerson&gt; response = restTemplate.exchange(URL, HttpMethod.DELETE, entity, ClientPerson.class, 3);
        assertEquals(HttpStatus.NO_CONTENT, response.getStatusCode());
        
        try {
            response = restTemplate.exchange(URL, HttpMethod.GET, entity, ClientPerson.class, 3);
            Assert.fail("404 error expected");
        } catch (HttpClientErrorException e) {
            assertEquals(HttpStatus.NOT_FOUND, e.getStatusCode());
        }
    }
}
</pre>

&nbsp;

## <span style="color: #0b5394;">5 Conclusion</span>

This second part of the tutorial has shown us how to implement a Spring Integration application with no XML configuration, using the new Spring Integration Java DSL. Although flow configuration is more readable using Java 8 lambdas, we still have the option to use Java DSL with previous versions of the language.

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.