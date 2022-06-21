---
id: 18
title: Exposing HTTP Restful API with Inbound Adapters. Part 1 (XML)
date: 2014-12-22T07:28:00+01:00
author: xpadro
layout: post
permalink: /2014/12/exposing-http-restful-api-with-inbound-adapters-part-1-xml.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2014/12/exposing-http-restful-api-with-inbound.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/3471462930649076522
image: /wp-content/uploads/2014/12/diagram.png
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

The purpose of this post is to implement an HTTP Restful API using Spring Integration <a href="http://docs.spring.io/spring-integration/reference/html/http.html" target="_blank" rel="noopener">HTTP</a> inbound adapters. This tutorial is divided into two parts:

  * XML configuration example (this same post).
  * Java DSL example. This will be explained in the next part of this tutorial, showing how to configure the application using <a href="https://github.com/spring-projects/spring-integration-java-dsl/wiki/Spring-Integration-Java-DSL-Reference" target="_blank" rel="noopener">Spring Integration Java DSL</a>, with examples with both Java 7 and Java 8.

Before looking at the code, let’s take a glance at the following diagram, which shows the different services exposed by the application:

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <img loading="lazy" class="alignnone wp-image-190 size-full" src="http://xpadro.com/wp-content/uploads/2014/12/diagram-1.png" alt="HTTP inbound adapters XML REST API diagram" width="920" height="716" srcset="https://xpadro.com/wp-content/uploads/2014/12/diagram-1.png 920w, https://xpadro.com/wp-content/uploads/2014/12/diagram-1-300x233.png 300w, https://xpadro.com/wp-content/uploads/2014/12/diagram-1-768x598.png 768w" sizes="(max-width: 920px) 100vw, 920px" />
</div>

&nbsp;

GET operations are handled by an HTTP inbound gateway, while the rest (PUT, POST and DELETE) are handled by HTTP inbound channel adapters, since no response body is sent back to the client. Each operation will be explained in the following sections:

  1. Introduction
  2. Application configuration
  3. Get operation
  4. Put and post operations
  5. Delete operation
  6. Conclusion

The source code is available at <a href="https://github.com/xpadro/spring-integration/tree/master/http/http-xml" target="_blank" rel="noopener">Github</a>.

## <span style="color: #0b5394;">1 Application configuration</span>

The [web.xml](https://github.com/xpadro/spring-integration/blob/master/int-http-xml/src/main/webapp/WEB-INF/web.xml) file contains the definition of the Dispatcher Servlet:

<pre class="lang:xhtml decode:true ">&lt;servlet&gt;
    &lt;servlet-name&gt;springServlet&lt;/servlet-name&gt;
    &lt;servlet-class&gt;org.springframework.web.servlet.DispatcherServlet&lt;/servlet-class&gt;
    &lt;init-param&gt;
        &lt;param-name&gt;contextConfigLocation&lt;/param-name&gt;
        &lt;param-value&gt;classpath:xpadro/spring/integration/configuration/http-inbound-config.xml&lt;/param-value&gt;
    &lt;/init-param&gt;
&lt;/servlet&gt;
&lt;servlet-mapping&gt;
    &lt;servlet-name&gt;springServlet&lt;/servlet-name&gt;
    &lt;url-pattern&gt;/spring/*&lt;/url-pattern&gt;
&lt;/servlet-mapping&gt;</pre>

&nbsp;

The [http-inbound-config.xml](https://github.com/xpadro/spring-integration/blob/master/int-http-xml/src/main/resources/xpadro/spring/integration/configuration/http-inbound-config.xml) file will be explained in the following sections.

The [pom.xml](https://github.com/xpadro/spring-integration/blob/master/int-http-xml/pom.xml) file is detailed below. It is important to note the jackson libraries. Since we will be using JSON to represent our resources, these libraries must be present in the class path. Otherwise, the framework won’t register the required converter.

<pre class="lang:xhtml decode:true">&lt;properties&gt;
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

## <span style="color: #0b5394;">2 Get operation</span>

The configuration of the flow is shown below:

<u>http-inbound-config.xml</u>

The gateway receives requests to this path: /persons/{personId}. Once a request has arrived, a message is created and sent to httpGetChannel channel. The gateway will then wait for a <a href="http://www.eaipatterns.com/MessagingAdapter.html" target="_blank" rel="noopener">service activator</a> (personEndpoint) to return a response:

<pre class="lang:xhtml decode:true ">&lt;int-http:inbound-gateway request-channel="httpGetChannel"
    reply-channel="responseChannel"
    supported-methods="GET" 
    path="/persons/{personId}"
    payload-expression="#pathVariables.personId"&gt;
    
    &lt;int-http:request-mapping consumes="application/json" produces="application/json"/&gt;
&lt;/int-http:inbound-gateway&gt;

&lt;int:service-activator ref="personEndpoint" method="get" input-channel="httpGetChannel" output-channel="responseChannel"/&gt;</pre>

&nbsp;

Now, some points need to be explained:

  * **supported-methods**: this attribute indicates which methods are supported by the gateway (only GET requests).
  * **payload-expression**: What we are doing here is getting the value from personId variable in the URI template and putting it in the message’s payload. For example, the request path ‘/persons/3’ will become a Message with a value ‘3’ as its payload.
  * **request-mapping**: We can include this element to specify several attributes and filter which requests will be mapped to the gateway. In the example, only requests that contain the value ‘application/json’ for Content-Type header (consumes attribute) and Accept header (produces attribute) will be handled by this gateway.

Once a request is mapped to this gateway, a message is built and sent to the service activator. In the example, we defined a simple bean that will get the required information from a service:

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
    
    //Other operations
}</pre>

&nbsp;

Depending on the response received from the service, we will return the requested person or a status code indicating that no person was found.

Now we will test that everything works as expected. First, we define a ClientPerson class to which the response will be converted:

<pre class="lang:java decode:true ">@JsonIgnoreProperties(ignoreUnknown = true)
public class ClientPerson implements Serializable {
    private static final long serialVersionUID = 1L;
    
    @JsonProperty("id")
    private int myId;
    private String name;
    
    public ClientPerson() {}
    
    public ClientPerson(int id, String name) {
        this.myId = id;
        this.name = name;
    }
    
    //Getters and setters
}</pre>

&nbsp;

Then we implement the test. The buildHeaders method is where we specify Accept and Content-Type headers. Remember that we restricted requests with ‘application/json’ values in those headers.

<pre class="lang:java decode:true ">@RunWith(BlockJUnit4ClassRunner.class)
public class GetOperationsTest {
    private static final String URL = "http://localhost:8081/int-http-xml/spring/persons/{personId}";
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
    
    @Test
    public void getResource_responseIsReceivedAsJson() {
        HttpEntity&lt;Integer&gt; entity = new HttpEntity&lt;&gt;(buildHeaders());
        ResponseEntity&lt;String&gt; response = restTemplate.exchange(URL, HttpMethod.GET, entity, String.class, 1);
        assertEquals("{\"id\":1,\"name\":\"John\",\"age\":25}", response.getBody());
        assertEquals(HttpStatus.OK, response.getStatusCode());
    }
    
    @Test(expected=HttpClientErrorException.class)
    public void getResource_sendXml_415errorReturned() {
        HttpHeaders headers = new HttpHeaders();
        headers.setAccept(Arrays.asList(MediaType.APPLICATION_JSON));
        headers.setContentType(MediaType.APPLICATION_XML);
        HttpEntity&lt;Integer&gt; entity = new HttpEntity&lt;&gt;(headers);
        restTemplate.exchange(URL, HttpMethod.GET, entity, ClientPerson.class, 1);
    }
    
    @Test(expected=HttpClientErrorException.class)
    public void getResource_expectXml_receiveJson_406errorReturned() {
        HttpHeaders headers = new HttpHeaders();
        headers.setAccept(Arrays.asList(MediaType.APPLICATION_XML));
        headers.setContentType(MediaType.APPLICATION_JSON);
        HttpEntity&lt;Integer&gt; entity = new HttpEntity&lt;&gt;(headers);
        restTemplate.exchange(URL, HttpMethod.GET, entity, ClientPerson.class, 1);
    }
    
    @Test(expected=HttpClientErrorException.class)
    public void getResource_resourceNotFound_404errorReturned() {
        HttpEntity&lt;Integer&gt; entity = new HttpEntity&lt;&gt;(buildHeaders());
        restTemplate.exchange(URL, HttpMethod.GET, entity, ClientPerson.class, 8);
    }
}</pre>

&nbsp;

Not specifying a correct value in the Content-Type header will result in a 415 Unsupported Media Type error, since the gateway does not support this media type.

On the other hand, specifying an incorrect value in the Accept header will result in a 406 Not Acceptable error, since the gateway is returning another type of content than the expected.

## <span style="color: #0b5394;">3 Put and post operations</span>

For PUT and POST operations, we are using the same HTTP inbound channel adapter, taking advantage of the possibility to define several paths and methods to it. Once a request arrives, a router will be responsible to delivering the message to the correct endpoint.

<u>http-inbound-config.xml </u>

<pre class="lang:xhtml decode:true ">&lt;int-http:inbound-channel-adapter channel="routeRequest" 
    status-code-expression="T(org.springframework.http.HttpStatus).NO_CONTENT"
    supported-methods="POST, PUT" 
    path="/persons, /persons/{personId}"
    request-payload-type="xpadro.spring.integration.server.model.ServerPerson"&gt;
    
    &lt;int-http:request-mapping consumes="application/json"/&gt;
&lt;/int-http:inbound-channel-adapter&gt;

&lt;int:router input-channel="routeRequest" expression="headers.http_requestMethod"&gt;
    &lt;int:mapping value="PUT" channel="httpPutChannel"/&gt;
    &lt;int:mapping value="POST" channel="httpPostChannel"/&gt;
&lt;/int:router&gt;

&lt;int:service-activator ref="personEndpoint" method="put" input-channel="httpPutChannel"/&gt;
&lt;int:service-activator ref="personEndpoint" method="post" input-channel="httpPostChannel"/&gt;</pre>

&nbsp;

This channel adapter includes two new attributes:

  * **status-code-expression**: By default, the channel adapter acknowledges that the request has been received and returns a 200 status code. If we want to override this behavior, we can specify a different status code in this attribute. Here, we specify that these operations will return a 204 No Content status code.
  * **request-payload-type**: This attribute specifies what class will the request body be converted to. If we do not define it, it will not be able to convert to the class that the service activator is expecting (ServerPerson).

When a request is received, the adapter sends it to the routeRequest channel, where a router is expecting it. This router will inspect the message headers and depending on the value of the ‘http_requestMethod’ header, it will deliver it to the appropriate endpoint.

Both PUT and POST operations are handled by the same bean:

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

Return type is void because no response is expected; the inbound adapter will handle the return of the status code.

[PutOperationsTest](https://github.com/xpadro/spring-integration/blob/master/int-http-xml/src/test/java/xpadro/spring/integration/test/PutOperationsTest.java) validates that the correct status code is returned and that the resource has been updated:

<pre class="lang:java decode:true ">@RunWith(BlockJUnit4ClassRunner.class)
public class PutOperationsTest {
    private static final String URL = "http://localhost:8081/int-http-xml/spring/persons/{personId}";
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

[PostOperationsTest](https://github.com/xpadro/spring-integration/blob/master/int-http-xml/src/test/java/xpadro/spring/integration/test/PostOperationsTest.java) validates that the new resource has been added:

<pre class="lang:java decode:true ">@RunWith(BlockJUnit4ClassRunner.class)
public class PostOperationsTest {
    private static final String POST_URL = "http://localhost:8081/int-http-xml/spring/persons";
    private static final String GET_URL = "http://localhost:8081/int-http-xml/spring/persons/{personId}";
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

The last operation of our restful API is the delete operation. This time we use a single channel adapter for this purpose:

<pre class="lang:xhtml decode:true ">&lt;int-http:inbound-channel-adapter channel="httpDeleteChannel" 
    status-code-expression="T(org.springframework.http.HttpStatus).NO_CONTENT"
    supported-methods="DELETE" 
    path="/persons/{personId}" 
    payload-expression="#pathVariables.personId"&gt;
    
    &lt;int-http:request-mapping consumes="application/json"/&gt;
&lt;/int-http:inbound-channel-adapter&gt;

&lt;int:service-activator ref="personEndpoint" method="delete" input-channel="httpDeleteChannel"/&gt;</pre>

&nbsp;

The channel adapter lets us define the returning status code and we are using the payload-expression attribute to map the requested personId to the message body. The configuration is a little bit different from  those in previous operations but there’s nothing not already explained here.

The service activator, our person endpoint, will request the person service to delete this resource.

<pre class="lang:java decode:true ">public void delete(Message&lt;String&gt; msg) {
    long id = Long.valueOf(msg.getPayload());
    service.deletePerson(id);
}</pre>

&nbsp;

Finally, the required test:

<pre class="lang:java decode:true ">@RunWith(BlockJUnit4ClassRunner.class)
public class DeleteOperationsTest {
    private static final String URL = "http://localhost:8081/int-http-xml/spring/persons/{personId}";
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
}</pre>

&nbsp;

## <span style="color: #0b5394;">5 Conclusion</span>

This post has been an introduction to our application in order to understand how it is structured from a known point of view (xml configuration). In the next part of this tutorial, we are going to implement this same application using Java DSL. The application will be configured to run with Java 8, but when lambdas are used, I will also show how it can be done with Java 7.

You can read the second part of this tutorial [here](http://xpadro.com/2014/12/exposing-http-restful-api-with-inbound_22.html).

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.