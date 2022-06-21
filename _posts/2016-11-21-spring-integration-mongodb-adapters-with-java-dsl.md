---
id: 7
title: Spring Integration MongoDB adapters with Java DSL
date: 2016-11-21T10:17:00+01:00
author: xpadro
layout: post
permalink: /2016/11/spring-integration-mongodb-adapters-with-java-dsl.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2016/11/spring-integration-mongodb-adapters.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/8893511362321124238
image: /wp-content/uploads/2016/11/flow_firstPart.png
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

This post explains how to save and retrieve entities from a MongoDB database using Spring Integration. In order to accomplish that, we are going to configure inbound and outbound MongoDB channel adapters using the Java DSL configuration extension. As an example, we are going to build an application to allow you to write orders to a MongoDB store, and then retrieve them for processing.

The application flow can be split in two parts:

  * New orders are sent to the messaging system, where they will be converted to actual products and then stored to MongoDB.

  * On the other hand, another component is continuously polling the database and processing any new product it finds.

The source code can be found in my <a href="https://github.com/xpadro/spring-integration/tree/master/mongodb/mongo-basic" target="_blank" rel="noopener">Spring Integration repository</a>.

## <span style="color: #0b5394;">1 MessagingGateway &#8211; Entering the messaging system</span>

Our application does not know anything about the messaging system. In fact, it will just create new orders and send them to an interface (OrderService):

<pre class="lang:java decode:true ">@SpringBootApplication
@EnableIntegration
public class MongodbBasicApplication {
    
    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(MongodbBasicApplication.class, args);
        new MongodbBasicApplication().start(context);
    }
    
    public void start(ConfigurableApplicationContext context) {
        resetDatabase(context);
        
        Order order1 = new Order("1", true);
        Order order2 = new Order("2", false);
        Order order3 = new Order("3", true);
        
        InfrastructureConfiguration.OrderService orderService = context.getBean(InfrastructureConfiguration.OrderService.class);
        
        orderService.order(order1);
        orderService.order(order2);
        orderService.order(order3);
    }
    
    private void resetDatabase(ConfigurableApplicationContext context) {
        ProductRepository productRepository = context.getBean(ProductRepository.class);
        productRepository.deleteAll();
    }
}</pre>

&nbsp;

Taking an initial look at the configuration, we can see that the OrderService is actually a messaging gateway.

<pre class="lang:java decode:true ">@Configuration
@ComponentScan("xpadro.spring.integration.endpoint")
@IntegrationComponentScan("xpadro.spring.integration.mongodb")
public class InfrastructureConfiguration {

    @MessagingGateway
    public interface OrderService {

        @Gateway(requestChannel = "sendOrder.input")
        void order(Order order);
    }
    
    ...
}</pre>

&nbsp;

Any order sent to the order method will be introduced to the messaging system as a Message<Order> through the &#8216;sendOrder.input&#8217; direct channel.

## <span style="color: #0b5394;">2 First part &#8211; processing orders</span>

The first part of the Spring Integration messaging flow is composed by the following components:

<div style="clear: both; text-align: center;">
  <img loading="lazy" class="alignnone wp-image-230 size-full" src="http://xpadro.com/wp-content/uploads/2016/11/flow_firstPart-1.png" alt="mongoDB channel adapters - processing order diagram" width="650" height="102" srcset="https://xpadro.com/wp-content/uploads/2016/11/flow_firstPart-1.png 650w, https://xpadro.com/wp-content/uploads/2016/11/flow_firstPart-1-300x47.png 300w" sizes="(max-width: 650px) 100vw, 650px" />
</div>

&nbsp;

We use a lambda to create an IntegrationFlow definition, which registers a DirectChannel as its input channel. The name of the input channel is resolved as &#8216;beanName + .input&#8217;. Hence, the name is the one we specified in the gateway: &#8216;sendOrder.input&#8217;

<pre class="lang:java decode:true ">@Bean
@Autowired
public IntegrationFlow sendOrder(MongoDbFactory mongo) {
    return f -&gt; f
        .transform(Transformers.converter(orderToProductConverter()))
        .handle(mongoOutboundAdapter(mongo));
}</pre>

&nbsp;

The first thing the flow does when receiving a new order is use a transformer to convert the order into a product. To register a transformer we can use the Transformers factory provided by the DSL API. Here, we have different possibilities. The one I chose is using a <a href="https://docs.spring.io/spring-integration/api/org/springframework/integration/transformer/PayloadTypeConvertingTransformer.html" target="_blank" rel="noopener">PayloadTypeConvertingTransformer</a>, which delegates to a converter the transformation of the payload into an object.

<pre class="lang:java decode:true ">public class OrderToProductConverter implements Converter&lt;Order, Product&gt; {

    @Override
    public Product convert(Order order) {
        return new Product(order.getId(), order.isPremium());
    }
}</pre>

&nbsp;

The next step in the orders flow is to store the newly created product to the database. Here, we use a MongoDB outbound adapter:

<pre class="lang:java decode:true ">@Bean
@Autowired
public MessageHandler mongoOutboundAdapter(MongoDbFactory mongo) {
    MongoDbStoringMessageHandler mongoHandler = new MongoDbStoringMessageHandler(mongo);
    mongoHandler.setCollectionNameExpression(new LiteralExpression("product"));
    return mongoHandler;
}</pre>

&nbsp;

If you wonder what the message handler is actually doing internally, it uses a mongoTemplate to save the entity:

<pre class="lang:java decode:true ">@Override
protected void handleMessageInternal(Message&lt;?&gt; message) throws Exception {
    String collectionName = this.collectionNameExpression.getValue(this.evaluationContext, message, String.class);
    Object payload = message.getPayload();
    
    this.mongoTemplate.save(payload, collectionName);
}</pre>

&nbsp;

## <span style="color: #0b5394;">3 Second part &#8211; processing products</span>

In this second part we have another integration flow for processing products:

<div style="clear: both; text-align: center;">
  <img loading="lazy" class="alignnone wp-image-231 size-full" src="http://xpadro.com/wp-content/uploads/2016/11/flow_secondPart-1.png" alt="mongoDB channel adapters - processing product diagram" width="650" height="243" srcset="https://xpadro.com/wp-content/uploads/2016/11/flow_secondPart-1.png 650w, https://xpadro.com/wp-content/uploads/2016/11/flow_secondPart-1-300x112.png 300w" sizes="(max-width: 650px) 100vw, 650px" />
</div>

&nbsp;

In order to retrieve previously created products, we have defined an inbound channel adapter which will continuously be polling the MongoDB database:

<pre class="lang:java decode:true ">@Bean
@Autowired
public IntegrationFlow processProduct(MongoDbFactory mongo) {
    return IntegrationFlows.from(mongoMessageSource(mongo), c -&gt; c.poller(Pollers.fixedDelay(3, TimeUnit.SECONDS)))
        .route(Product::isPremium, this::routeProducts)
        .handle(mongoOutboundAdapter(mongo))
        .get();
}</pre>

&nbsp;

The MongoDB inbound channel adapter is the one responsible for polling products from the database. We specify the query in the constructor. In this case, we poll one non processed product each time:

<pre class="lang:java decode:true">@Bean
@Autowired
public MessageSource&lt;Object&gt; mongoMessageSource(MongoDbFactory mongo) {
    MongoDbMessageSource messageSource = new MongoDbMessageSource(mongo, new LiteralExpression("{'processed' : false}"));
    messageSource.setExpectSingleResult(true);
    messageSource.setEntityClass(Product.class);
    messageSource.setCollectionNameExpression(new LiteralExpression("product"));
    
    return messageSource;
}</pre>

&nbsp;

The router definition shows how the product is sent to a different service activator method depending on the &#8216;premium&#8217; field:

<pre class="lang:java decode:true ">private RouterSpec&lt;Boolean, MethodInvokingRouter&gt; routeProducts(RouterSpec&lt;Boolean, MethodInvokingRouter&gt; mapping) {
    return mapping
        .subFlowMapping(true, sf -&gt; sf.handle(productProcessor(), "fastProcess"))
        .subFlowMapping(false, sf -&gt; sf.handle(productProcessor(), "process"));
}</pre>

&nbsp;

As a service activator, we have a simple bean which logs a message and sets the product as processed. Then, it will return the product so it can be handled by the next endpoint in the flow.

<pre class="lang:java decode:true ">public class ProductProcessor {

    public Product process(Product product) {
        return doProcess(product, String.format("Processing product %s", product.getId()));
    }

    public Product fastProcess(Product product) {
        return doProcess(product, String.format("Fast processing product %s", product.getId()));
    }

    private Product doProcess(Product product, String message) {
        System.out.println(message);
        product.setProcessed(true);
        return product;
    }
}</pre>

&nbsp;

The reason for setting the product as processed is because the next step is to update its status in the database in order to not poll it again. We save it by redirecting the flow to the mongoDb outbound channel adapter again.

## <span style="color: #0b5394;">4 Conclusion</span>

You have seen what endpoints you do have to use in order to interact with a MongoDB database using Spring Integration. The outbound channel adapter passively saves products to the database, while the inbound channel adapter actively polls the database to retrieve new products.

If you found this post useful, please share it or star my repository. I appreciate it 🙂

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.

&nbsp;