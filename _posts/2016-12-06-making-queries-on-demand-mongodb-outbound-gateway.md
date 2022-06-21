---
id: 6
title: 'Making queries on demand: MongoDB outbound gateway'
date: 2016-12-06T09:19:00+01:00
author: xpadro
layout: post
permalink: /2016/12/making-queries-on-demand-mongodb-outbound-gateway.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2016/12/making-queries-on-demand-mongodb.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/379609152264140622
categories:
  - Integration
  - mongoDB
  - Spring
  - Spring-Boot
tags:
  - Integration
  - MongoDB
  - Spring
  - Spring Boot
---

In order to read data from MongoDb, Spring Integration comes with the MongoDb inbound channel adapter. This adapter uses a poller to continuously retrieve documents from the database. However, sometimes we may need to query the database on demand, based on the result of another endpoint.

Taking advantage of Spring&#8217;s extensibility, I implemented a MongoDb outbound gateway. The purpose of this gateway is to react to some request, make a query to the database and return the result.

In order to show you how the gateway works, I will use a simple example and modify it to implement the gateway with different configurations.

This example consists in a messaging gateway as the entry point to the integration flow. Once a message enters the flow, the mongoDb outbound gateway makes a query to the database and the result is then sent to another channel where a service activator will process it.

The source code for these examples and the gateway implementation can be found in <a href="https://github.com/xpadro/spring-integration/tree/master/mongodb" target="_blank" rel="noopener">my repository</a>.

## <span style="color: #0b5394;">1 Java DSL example</span>

I implemented the <a href="https://github.com/xpadro/spring-integration/blob/master/mongodb/mongo-gateway/src/main/java/xpadro/spring/integration/mongodb/gateway/MongoDb.java" target="_blank" rel="noopener">MongoDb</a> static class to ease the definition of the gateway. I took the idea from the Spring Integration <a href="http://docs.spring.io/autorepo/docs/spring-integration-java-dsl/1.2.0.BUILD-SNAPSHOT/api/org/springframework/integration/dsl/jpa/Jpa.html" target="_blank" rel="noopener">Jpa</a> class.

In the following configuration you can see the flow _requestPerson_. An invocation to PersonService&#8217;s _send_ method will send a message to the flow, where the mongoDb outbound gateway will then query the database with a pre-defined query ({id : 1}):

<pre class="lang:java decode:true ">@ComponentScan("xpadro.spring.integration.mongodb")
@IntegrationComponentScan("xpadro.spring.integration.mongodb")
public class JavaDSLQueryConfiguration {
    
    @MessagingGateway
    public interface PersonService {
        
        @Gateway(requestChannel = "requestPerson.input")
        void send(RequestMessage requestMessage);
    }

    @Bean
    @Autowired
    public IntegrationFlow requestPerson(MongoDbFactory mongo) {
        return f -&gt; f
                .handle(outboundGateway(mongo))
                .handle(resultHandler(), "handle");
    }

    @Bean
    public ResultHandler resultHandler() {
        return new ResultHandler();
    }

    private MongoDbOutboundGatewaySpec outboundGateway(MongoDbFactory mongo) {
        return MongoDb.outboundGateway(mongo)
                .query("{id: 1}")
                .collectionNameExpression(new LiteralExpression("person"))
                .expectSingleResult(true)
                .entityClass(Person.class);
    }
}</pre>

&nbsp;

The result handler is a &#8220;very useful&#8221; component which will log the retrieved person:

<pre class="lang:java decode:true ">public class ResultHandler {

    public void handle(Person person) {
        System.out.println(String.format("Person retrieved: %s", person));
    }
}</pre>

&nbsp;

In order to start the flow, the following application sends a message to the PersonService gateway:

<pre class="lang:java decode:true ">@SpringBootApplication
@EnableIntegration
@Import(JavaDSLQueryConfiguration.class)
public class JavaDSLQueryApplication extends AbstractApplication {

	public static void main(String[] args) {
		ConfigurableApplicationContext context = SpringApplication.run(JavaDSLQueryApplication.class, args);
		new JavaDSLQueryApplication().start(context);
	}

	public void start(ConfigurableApplicationContext context) {
		resetDatabase(context);

		JavaDSLQueryConfiguration.PersonService personService = context.getBean(JavaDSLQueryConfiguration.PersonService.class);
		personService.send(new RequestMessage());
	}
}</pre>

&nbsp;

As a note, the abstract class just contains the logic to set up the database, which is used along all the other examples.

## <span style="color: #0b5394;">2 Java DSL example with dynamic query expression</span>

The previous example was useful to see how to define the gateway, but having a hardcoded query may not be the most used case.

In this example, the query is defined in the message sent to the integration flow:

<pre class="lang:java decode:true ">personService.send(new RequestMessage("{id : 2}"));</pre>

&nbsp;

In the configuration file, the gateway&#8217;s _queryExpression_ property resolves the query dynamically by retrieving the _data_ property of the message&#8217;s payload:

<pre class="lang:java decode:true">private MongoDbOutboundGatewaySpec outboundGateway(MongoDbFactory mongo) {
        return MongoDb.outboundGateway(mongo)
                .queryExpression("payload.data")
                .collectionNameExpression(new LiteralExpression("person"))
                .expectSingleResult(true)
                .entityClass(Person.class);
    }</pre>

&nbsp;

## <span style="color: #0b5394;">3 Java DSL example returning multiple results</span>

The two previous examples retrieved a single document from the database. In this next example, the query returns a list with all documents matching the query:

In the request message we specify the query to find all documents in the persons collection:

<pre class="lang:java decode:true ">personService.send(new RequestMessage("{}"));</pre>

&nbsp;

In the configuration, we have to remove the _expectSingleResult_ property from the gateway (or set it to false). Additionally, we can specify a limit:

<pre class="lang:java decode:true ">private MongoDbOutboundGatewaySpec outboundGateway(MongoDbFactory mongo) {
        return MongoDb.outboundGateway(mongo)
                .queryExpression("payload.data")
                .collectionNameExpression(new LiteralExpression("person"))
                .entityClass(Person.class)
                .maxResults(2);
    }</pre>

&nbsp;

Finally, we have to define another method in the ResultHandler class to handle multiple results:

<pre class="lang:java decode:true">public void handle(List&lt;Person&gt; persons) {
        String names = persons.stream().map(Person::getName).collect(Collectors.joining(", "));
        System.out.println(String.format("Persons retrieved: %s", names));
    }</pre>

&nbsp;

## <span style="color: #0b5394;">4 Java Config example</span>

In this last example, Java Config is used instead of Java DSL to configure the whole flow. On the application&#8217;s side everything is the same. We just query the person service for a specific document:

<pre class="lang:java decode:true ">personService.send(new RequestMessage("{id : 3}"));</pre>

&nbsp;

When using Java Config, we have to build the <a href="https://github.com/xpadro/spring-integration/blob/master/mongodb/mongo-gateway/src/main/java/xpadro/spring/integration/mongodb/gateway/MongoDbExecutor.java" target="_blank" rel="noopener">MongoDbExecutor</a>, which is used by the gateway to do the queries.

<pre class="lang:java decode:true ">@ComponentScan("xpadro.spring.integration.mongodb")
@IntegrationComponentScan("xpadro.spring.integration.mongodb")
public class JavaConfigQueryConfiguration {

    @MessagingGateway
    public interface PersonService {

        @Gateway(requestChannel = "personInput")
        void send(RequestMessage requestMessage);
    }

    @Bean
    public ResultServiceActivator resultHandler() {
        return new ResultServiceActivator();
    }

    @Bean
    @ServiceActivator(inputChannel = "personInput")
    public MessageHandler mongodbOutbound(MongoDbFactory mongo) {
        MongoDbExecutor mongoDbExecutor = new MongoDbExecutor(mongo);
        mongoDbExecutor.setCollectionNameExpression(new LiteralExpression("person"));
        mongoDbExecutor.setMongoDbQueryExpression("payload.data");
        mongoDbExecutor.setExpectSingleResult(true);
        mongoDbExecutor.setEntityClass(Person.class);

        MongoDbOutboundGateway gateway = new MongoDbOutboundGateway(mongoDbExecutor);
        gateway.setOutputChannelName("personOutput");

        return gateway;
    }
}</pre>

&nbsp;

Listening to the gateway&#8217;s output channel, we define a service activator to handle the retrieved person:

<pre class="lang:java decode:true">public class ResultServiceActivator {

    @ServiceActivator(inputChannel = "personOutput")
    public void handle(Person person) {
        System.out.println(String.format("Person retrieved: %s", person));
    }
}</pre>

&nbsp;

## <span style="color: #0b5394;">5 Conclusion</span>

An outbound gateway is suitable when you need to do queries on demand instead of actively polling the database. Currently, this implementation supports setup with Java Config and Java DSL. For now, I haven&#8217;t implemented the parsers needed to support XML configuration since I think these two ways of configuration cover the main necessity.

If you found this post useful, please share it or star my repository 🙂

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.

&nbsp;