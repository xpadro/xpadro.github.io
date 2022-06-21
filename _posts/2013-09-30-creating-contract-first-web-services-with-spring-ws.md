---
id: 32
title: Creating contract-first web services with Spring WS
date: 2013-09-30T16:18:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/09/30/creating-contract-first-web-services-with-spring-ws/
permalink: /2013/09/creating-contract-first-web-services-with-spring-ws.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/09/creating-contract-first-web-services_30.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/6296360888380705889
categories:
  - Spring
  - Spring WS
tags:
  - Spring
  - web services
---

This article explains how to implement and test SOAP web services with <a href="http://projects.spring.io/spring-ws/" target="_blank" rel="noopener">Spring WS project</a>. This example uses JAXB2 for (un)marshalling. To develop the service, I’ll use the contract-first approach, which consists in defining the service contract first, and based on this contract implement the service.

The article is divided into the following sections:

<div style="margin-left: 20px;">
  2   Explaining the application
</div>

<div style="margin-left: 20px;">
  3   Implementing the service
</div>

<div style="margin-left: 60px;">
  3.1   Creating the contract
</div>

<div style="margin-left: 60px;">
  3.2   Generating Java classes
</div>

<div style="margin-left: 60px;">
  3.3   Implementing the SOAP endpoint
</div>

<div style="margin-left: 60px;">
  3.4   Configuring the application
</div>

<div style="margin-left: 20px;">
  4   Testing the service
</div>

<div style="margin-left: 20px;">
  5   Additional information
</div>

<div style="margin-left: 60px;">
  5.1   Implementing the client
</div>

<div style="margin-left: 60px;">
  5.2   How it works internally
</div>

&nbsp;

## <span style="color: #0b5394;">1 Explaining the application</span>

The example application processes orders. We’ve got a front controller (messageDispatcher servlet) that will handle order requests, invoke the service to process the order and return a result.

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://1.bp.blogspot.com/-hCo0pGs4kKw/VUZJXgJxaKI/AAAAAAAAEmY/e22uHyN-now/s1600/aplicacio.png"><img loading="lazy" class="alignnone" src="https://1.bp.blogspot.com/-hCo0pGs4kKw/VUZJXgJxaKI/AAAAAAAAEmY/e22uHyN-now/s1600/aplicacio.png" alt="application using web services with Spring WS" width="400" height="163" border="0" /></a>
</div>

You can get the source code at <a href="https://github.com/xpadro/spring-integration/tree/master/webservices/spring-ws" target="_blank" rel="noopener">my Github repository</a>.

### <span style="color: #0b5394;">2 Implementing the service<br /> 3.1 Creating the contract</span>

Since we will use the contract-first approach, the simplest way to create the contract is by first defining sample xml documents and from that, we will generate the contract using a tool. Below are the sample xml documents:

#### <span style="color: #003366;">client-request.xml</span>

<pre class="lang:xhtml decode:true ">&lt;clientDataRequest xmlns="http://www.xpadro.spring.samples.com/orders"
    clientId="A-123"
    productId="C5FH"
    quantity="5" /&gt;
view raw</pre>

&nbsp;

#### <span style="color: #003366;">client-response.xml</span>

<pre class="lang:xhtml decode:true ">&lt;clientDataResponse xmlns="http://www.xpadro.spring.samples.com/orders" 
    confirmationId="7890B"
    orderDate="2013-09-22"
    amount="15.50" /&gt;</pre>

&nbsp;

In order to create the schema, we can use Trang, which is an open source tool that will allow us to generate the xsd schema from the xml documents. I’ve included this library into the project build path (you can get this jar from Trang <a href="http://code.google.com/p/jing-trang/downloads/list" target="_blank" rel="noopener">web site</a>) and I’ve created an Ant task for executing the conversion:

#### <span style="color: #003366;">generate-schema.xml</span>

<pre class="lang:xhtml decode:true ">&lt;project name="Ant-Generate-Schema-With-Trang" default="generate" basedir="."&gt;
    &lt;property name="src.dir" location="src" /&gt;
    &lt;property name="trang.dir" location="lib" /&gt;
    &lt;property name="source.dir" location="${src.dir}/main/webapp/WEB-INF/schemas/samples" /&gt;
    &lt;property name="schema.dir" location="${src.dir}/main/webapp/WEB-INF/schemas/xsd" /&gt;
    
    &lt;target name="generate" description="generates order schema"&gt;
        &lt;delete dir="${schema.dir}" /&gt;
        &lt;mkdir dir="${schema.dir}" /&gt;
        
        &lt;java jar="${trang.dir}/trang.jar" fork="true"&gt;
            &lt;arg value="${source.dir}/client-request.xml" /&gt;
            &lt;arg value="${schema.dir}/client-request.xsd" /&gt;
        &lt;/java&gt;
        
        &lt;java jar="${trang.dir}/trang.jar" fork="true"&gt;
            &lt;arg value="${source.dir}/client-response.xml" /&gt;
            &lt;arg value="${schema.dir}/client-response.xsd" /&gt;
        &lt;/java&gt;
    &lt;/target&gt;
&lt;/project&gt;</pre>

&nbsp;

Once the Ant task is executed, it will generate the schemas. Since schemas have been automatically generated, it is possible that we need to make some modifications to adapt it to our needs. Let’s take a look:

&nbsp;

#### <span style="color: #003366;">client-request.xsd</span>

<pre class="lang:xhtml decode:true ">&lt;xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema" 
    elementFormDefault="qualified" 
    targetNamespace="http://www.xpadro.spring.samples.com/orders" 
    xmlns:orders="http://www.xpadro.spring.samples.com/orders"&gt;
    
    &lt;xs:element name="clientDataRequest"&gt;
        &lt;xs:complexType&gt;
            &lt;xs:attribute name="clientId" use="required" type="xs:NCName"/&gt;
            &lt;xs:attribute name="productId" use="required" type="xs:NCName"/&gt;
            &lt;xs:attribute name="quantity" use="required" type="xs:integer"/&gt;
        &lt;/xs:complexType&gt;
    &lt;/xs:element&gt;
&lt;/xs:schema&gt;</pre>

&nbsp;

#### <span style="color: #003366;">client-response.xsd</span>

<pre class="lang:xhtml decode:true ">&lt;xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema" 
    elementFormDefault="qualified" 
    targetNamespace="http://www.xpadro.spring.samples.com/orders" 
    xmlns:orders="http://www.xpadro.spring.samples.com/orders"&gt;
    
    &lt;xs:element name="clientDataResponse"&gt;
        &lt;xs:complexType&gt;
            &lt;xs:attribute name="amount" use="required" type="xs:decimal"/&gt;
            &lt;xs:attribute name="confirmationId" use="required" type="xs:NMTOKEN"/&gt;
            &lt;xs:attribute name="orderDate" use="required" type="xs:NMTOKEN"/&gt;
        &lt;/xs:complexType&gt;
    &lt;/xs:element&gt;
&lt;/xs:schema&gt;</pre>

&nbsp;

We can add different validations to these schemas, but in this example I’ll just modify several types like clientId, productId and confirmationId (xs:string) and orderDate (xs:date). The mapping of XML data types to Java types is done by JAXB. You can check which are the mappings provided at <a href="http://docs.oracle.com/javaee/6/tutorial/doc/bnazc.html" target="_blank" rel="noopener">Oracle JavaEE tutorial</a>.

To finish with the schema, we will copy the response element into the request schema. I’ve created a third schema with both response and request:

&nbsp;

#### <span style="color: #003366;">client-service.xsd</span>

<pre class="lang:xhtml decode:true ">&lt;xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
    elementFormDefault="qualified" targetNamespace="http://www.xpadro.spring.samples.com/orders"
    xmlns:orders="http://www.xpadro.spring.samples.com/orders"&gt;
    
    &lt;xs:element name="clientDataRequest"&gt;
        &lt;xs:complexType&gt;
            &lt;xs:attribute name="clientId" use="required" type="xs:string" /&gt;
            &lt;xs:attribute name="productId" use="required" type="xs:string" /&gt;
            &lt;xs:attribute name="quantity" use="required" type="xs:integer" /&gt;
        &lt;/xs:complexType&gt;
    &lt;/xs:element&gt;
    
    &lt;xs:element name="clientDataResponse"&gt;
        &lt;xs:complexType&gt;
            &lt;xs:attribute name="amount" use="required" type="xs:decimal" /&gt;
            &lt;xs:attribute name="confirmationId" use="required" type="xs:string" /&gt;
            &lt;xs:attribute name="orderDate" use="required" type="xs:date" /&gt;
        &lt;/xs:complexType&gt;
    &lt;/xs:element&gt;
&lt;/xs:schema&gt;</pre>

&nbsp;

The last step would consist in writing the contract, generally expressed as a WSDL file. If you don’t want to create it by hand, the Spring-ws project provides us with a way to generate this file from an XSD schema. We will use this second approach as you will see in the _configuring the application_ section.

### <span style="color: #0b5394;">2.2   Generating Java classes</span>

We will use JAXB2 to generate request and response objects. The XJC compiler from JAXB will be responsible of converting these objects from the XSD schema that we generated before. It will be executed as an Ant task:

<pre class="lang:xhtml decode:true ">&lt;project name="Ant-Generate-Classes-With-JAXB2" default="generate" basedir="."&gt;
    &lt;property name="src.dir" location="src" /&gt;
    &lt;property name="java.dir" location="src/main/java" /&gt;
    &lt;property name="schema.dir" location="${src.dir}/main/webapp/WEB-INF/schemas/xsd" /&gt;
    
    &lt;target name="generate"&gt;
        &lt;exec executable="xjc"&gt;
            &lt;arg line=" -d ${java.dir} -p xpadro.spring.ws.types ${schema.dir}/client-service.xsd" /&gt;
        &lt;/exec&gt;
    &lt;/target&gt;
&lt;/project&gt;</pre>

This task will create Java classes in the xpadro.spring.ws.types package (you may need to refresh the project).

<div style="clear: both; text-align: center;">
  <a href="http://1.bp.blogspot.com/-z-6ogc-y0J8/VUZJYEWRkaI/AAAAAAAAEmg/gSTWGVy1wH8/s1600/types.png"><img loading="lazy" class="alignnone" src="https://1.bp.blogspot.com/-z-6ogc-y0J8/VUZJYEWRkaI/AAAAAAAAEmg/gSTWGVy1wH8/s1600/types.png" alt="created java types package structure" width="189" height="87" border="0" /></a>
</div>

<div style="clear: both; text-align: center;">
</div>

### <span style="color: #0b5394;">2.3   Implementing the SOAP endpoint</span>

The endpoint receives the unmarshalled message payload and uses this data to invoke the order service. It will then return the service response, which will be marshalled by the endpoint adapter:

<pre class="lang:java decode:true ">@Endpoint
public class OrderEndpoint {
    @Autowired
    private OrderService orderService;
    
    @PayloadRoot(localPart="clientDataRequest", namespace="http://www.xpadro.spring.samples.com/orders")
    public @ResponsePayload ClientDataResponse order(@RequestPayload ClientDataRequest orderData) {
        OrderConfirmation confirmation = 
            orderService.order(orderData.getClientId(), orderData.getProductId(), orderData.getQuantity().intValue());
        
        ClientDataResponse response = new ClientDataResponse();
        response.setConfirmationId(confirmation.getConfirmationId());
        BigDecimal amount = new BigDecimal(Float.toString(confirmation.getAmount()));
        response.setAmount(amount);
        response.setOrderDate(convertDate(confirmation.getOrderDate()));
        
        return response;
    }
    
    //date conversion
}</pre>

&nbsp;

Here&#8217;s a brief description of the annotations used by the endpoint:

**@Endpoint**: Registers the class as a component. In this way, the class will be detected by component scan.

**@PayloadRoot**: Registers the endpoint method as a handler for a request. This annotation will define what type of request message can be handled by the method. In our example, it will receive messages where its payload root element has the same namespace as defined in the XSD schema we created, and its local name is the one defined for the request (clientDataRequest).

**@RequestPayload**: Indicates the payload of the request message to be passed as a parameter to the method.

**@ResponsePayload**, indicates that the return value is used as the payload of the response message.

### <span style="color: #0b5394;">2.4   Configuring the application</span>

#### <span style="color: #003366;">web.xml</span>

Application configuration (like datasource, transactionManager&#8230;)

<pre class="lang:xhtml decode:true">&lt;context-param&gt;
    &lt;param-name&gt;contextConfigLocation&lt;/param-name&gt;
    &lt;param-value&gt;classpath:xpadro/spring/ws/config/root-config.xml&lt;/param-value&gt;
&lt;/context-param&gt;</pre>

&nbsp;

Loads the application context:

<pre class="lang:xhtml decode:true ">&lt;listener&gt;
    &lt;listener-class&gt;org.springframework.web.context.ContextLoaderListener&lt;/listener-class&gt;
&lt;/listener&gt;</pre>

&nbsp;

This is the servlet that will act as a Front Controller to handle all SOAP calls. Its function is to derive incoming XML messages to endpoints, much like the DispatcherServlet of Spring MVC:

<pre class="lang:xhtml decode:true ">&lt;servlet&gt;
    &lt;servlet-name&gt;orders&lt;/servlet-name&gt;
    &lt;servlet-class&gt;org.springframework.ws.transport.http.MessageDispatcherServlet&lt;/servlet-class&gt;
    &lt;init-param&gt;
        &lt;param-name&gt;contextConfigLocation&lt;/param-name&gt;
        &lt;param-value&gt;classpath:xpadro/spring/ws/config/servlet-config.xml&lt;/param-value&gt;
    &lt;/init-param&gt;
    &lt;load-on-startup&gt;1&lt;/load-on-startup&gt;
&lt;/servlet&gt;

&lt;servlet-mapping&gt;
    &lt;servlet-name&gt;orders&lt;/servlet-name&gt;
    &lt;url-pattern&gt;/orders/*&lt;/url-pattern&gt;
&lt;/servlet-mapping&gt;</pre>

&nbsp;

#### <span style="color: #003366;">servlet-config.xml</span>

This configuration contains web service infrastructure beans.

<pre class="lang:xhtml decode:true ">&lt;!-- Detects @Endpoint since it is a specialization of @Component --&gt;
&lt;context:component-scan base-package="xpadro.spring.ws"/&gt;

&lt;!-- detects @PayloadRoot --&gt;
&lt;ws:annotation-driven/&gt;

&lt;ws:dynamic-wsdl id="orderDefinition" portTypeName="Orders" locationUri="http://localhost:8081/spring-ws"&gt;
    &lt;ws:xsd location="/WEB-INF/schemas/xsd/client-service.xsd"/&gt;
&lt;/ws:dynamic-wsdl&gt;</pre>

&nbsp;

In the dynamic wsdl, it doesn’t matter what value you put in the locationUri attribute because it will be handled by the MessageDispatcherServlet. Hence, the wsdl will be available at:

http://localhost:8081/spring-ws/orders/whatever/orderDefinition.wsdl

## <span style="color: #0b5394;">3 Testing the service</span>

The following example creates a mock client which will access the web service:

<pre class="lang:java decode:true ">@ContextConfiguration("classpath:xpadro/spring/ws/test/config/test-server-config.xml")
@RunWith(SpringJUnit4ClassRunner.class)
public class TestWebService {
    @Autowired
    ApplicationContext context;
    
    private MockWebServiceClient mockClient;
    
    @Test
    public void testValidOrderRequest() {
        Source requestPayload = new StringSource(
            "&lt;clientDataRequest xmlns='http://www.xpadro.spring.samples.com/orders' " +
            "clientId='123' productId='XA-55' quantity='5'/&gt;");
        
        Source responsePayload = new StringSource(
            "&lt;clientDataResponse xmlns='http://www.xpadro.spring.samples.com/orders' " +
            "amount='55.99' confirmationId='GHKG34L' orderDate='2013-10-26+02:00'/&gt;");
        
        RequestCreator creator = RequestCreators.withPayload(requestPayload);
        mockClient = MockWebServiceClient.createClient(context);
        mockClient.sendRequest(creator).andExpect(ResponseMatchers.payload(responsePayload));
    }
    
    @Test
    public void testInvalidOrderRequest() {
        Source requestPayload = new StringSource(
            "&lt;clientDataRequest xmlns='http://www.xpadro.spring.samples.com/orders' " +
            "clientId='456' productId='XA-55' quantity='5'/&gt;");
        
        Source responsePayload = new StringSource(
            "&lt;SOAP-ENV:Fault xmlns:SOAP-ENV='http://schemas.xmlsoap.org/soap/envelope/'&gt;" +
            "&lt;faultcode&gt;SOAP-ENV:Server&lt;/faultcode&gt;&lt;faultstring xml:lang='en'&gt;Client [456] not found&lt;/faultstring&gt;&lt;/SOAP-ENV:Fault&gt;");
        
        RequestCreator creator = RequestCreators.withPayload(requestPayload);
        mockClient = MockWebServiceClient.createClient(context);
        mockClient.sendRequest(creator).andExpect(ResponseMatchers.payload(responsePayload));
    }
}</pre>

&nbsp;

The configuration file used on this test is pretty simple, just contains scanning of the service components:

<pre class="lang:xhtml decode:true ">&lt;context:component-scan base-package="xpadro.spring.ws"/&gt;
&lt;ws:annotation-driven/&gt;</pre>

&nbsp;

## <span style="color: #0b5394;">4 Additional information</span>

###  <span style="color: #0b5394;">4.1 Implementing a client</span>

To facilitate the client to access the web service, Spring provides us with the WebServiceTemplate class. This class contains methods for sending and receiving messages and it also uses converters to (un)marshal objects.

I’ve created a test that acts as a client of the service:

<pre class="lang:java decode:true ">@ContextConfiguration("classpath:xpadro/spring/ws/test/config/client-config.xml")
@RunWith(SpringJUnit4ClassRunner.class)
public class TestClient {
    @Autowired 
    WebServiceTemplate wsTemplate;
    
    @Test
    public void invokeOrderService() throws Exception {
        ClientDataRequest request = new ClientDataRequest();
        request.setClientId("123");
        request.setProductId("XA-55");
        request.setQuantity(new BigInteger("5", 10));
        
        ClientDataResponse response = (ClientDataResponse) wsTemplate.marshalSendAndReceive(request);
        
        assertNotNull(response);
        assertEquals(new BigDecimal("55.99"), response.getAmount());
        assertEquals("GHKG34L", response.getConfirmationId());
    }
}</pre>

&nbsp;

The configuration test file contains WebServiceTemplate configuration:

<pre class="lang:xhtml decode:true ">&lt;oxm:jaxb2-marshaller id="marshaller" contextPath="xpadro.spring.ws.types"/&gt;

&lt;bean class="org.springframework.ws.client.core.WebServiceTemplate"&gt;
    &lt;property name="marshaller" ref="marshaller" /&gt;
    &lt;property name="unmarshaller" ref="marshaller" /&gt;
    &lt;property name="defaultUri" value="http://localhost:8081/spring-ws/orders" /&gt; 
&lt;/bean&gt;</pre>

&nbsp;

Just remember to start the server with the deployed web service application before executing this test.

## <span style="color: #0b5394;">4.2 How it works internally</span>

If you just want to implement a web service, the article finished in the previous section. For those curious about how this really works, I will try to explain how a request is mapped to the endpoint, just a little more low-level than explained until this point.

When a request arrives to the MessageDispatcher, it relies on two components:

  1. It asks the EndpointMapping which is the appropriate endpoint.
  2. With the information received from the mapping it uses an endpoint adapter to invoke the endpoint. The adapter also support argument resolvers and return type handlers.

&nbsp;

#### <span style="color: #003366;">Endpoint mapping</span>

MessageDispatcher contains a list of endpoint mappings, each of them, containing a map of previously registered method endpoints. In our case, the JAXB mapping PayloadRootAnnotationMethodEndpointMapping has registered all methods annotated with @PayloadRoot. If the qualified name of the payload of the message resolves as a registered method, it will be returned to the MessageDispatcher. If we didn’t annotate our method it would fail to process the request.

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://3.bp.blogspot.com/-QnSge8ABUPA/VUZJXqgx1AI/AAAAAAAAEmQ/mNixsGFTFhc/s1600/sequencia1.png"><img loading="lazy" class="alignnone" src="https://3.bp.blogspot.com/-QnSge8ABUPA/VUZJXqgx1AI/AAAAAAAAEmQ/mNixsGFTFhc/s1600/sequencia1.png" alt="endpoint mapping diagram" width="320" height="172" border="0" /></a>
</div>

<div style="clear: both; text-align: center;">
</div>

#### <span style="color: #003366;">Endpoint adapter</span>

MessageDispatcher will then ask each of its endpoint adapters if it supports the current request. In our case, the adapter checks if the following conditions are both true:

  * At least one of the arguments passed to the method is annotated with @RequestPayload
  * If the endpoint method returns a response, it must be annotated with @ResponsePayload

If an adapter is returned, it will then invoke the endpoint, unmarshalling the parameter before passing it to the method. When the method returns a response, the adapter will marshal it.

The following diagram is a much reduced version of this step in order to keep it simple:

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://2.bp.blogspot.com/-PVXBOXJd104/VUZJXonslzI/AAAAAAAAEmU/XXvQ0WpuigY/s1600/sequencia2.png"><img loading="lazy" class="alignnone" src="https://2.bp.blogspot.com/-PVXBOXJd104/VUZJXonslzI/AAAAAAAAEmU/XXvQ0WpuigY/s1600/sequencia2.png" alt="endpoint adapter diagram" width="400" height="343" border="0" /></a>
</div>

<div style="clear: both; text-align: center;">
</div>

## <span style="color: #0b5394;">Conclusion</span>

We&#8217;ve seen an introduction on how to implement a simple web service and then test it. If you are interested, you can also take a look at <a href="http://docs.spring.io/spring-ws/site/reference/html/client.html#d5e1886" target="_blank" rel="noopener">how to test</a> the client-side with MockWebServiceServer.

&nbsp;