---
id: 30
title: How error handling works in Spring Integration
date: 2013-11-30T16:12:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/11/30/how-error-handling-works-in-spring-integration/
permalink: /2013/11/how-error-handling-works-in-spring-integration.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/11/how-error-handling-works-in-spring.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/4972227849072664767
categories:
  - Integration
  - Spring
tags:
  - Integration
  - Spring
  - Test
---

The target of this post is to show you how error handling works in Spring Integration, using the messaging system. You will see that error handling is different between synchronous and asynchronous messaging. As usual, I&#8217;ll skip the chat and proceed with some examples.

You can get the source code at <a href="https://github.com/xpadro/spring-integration/tree/master/generic/int-error">my Github repository</a>

## <span style="color: #0b5394;">1 The sample application </span>

I will use a basic example, since I want to focus on exception handling. The application consists in an order service. It receives an order, processes it and returns a confirmation.

Below we can see how the messaging system is configured:

<u>int-config.xml</u>

<pre class="lang:xhtml decode:true ">&lt;context:component-scan base-package="xpadro.spring.integration"/&gt;

&lt;int:gateway default-request-channel="requestChannel" service-interface="xpadro.spring.integration.service.OrderService"/&gt;

&lt;int:channel id="requestChannel"/&gt;

&lt;int:router input-channel="requestChannel" ref="orderRouter" method="redirectOrder"/&gt;

&lt;int:channel id="syncChannel"/&gt;

&lt;int:channel id="asyncChannel"&gt;
    &lt;int:queue capacity="5"/&gt;
&lt;/int:channel&gt;

&lt;int:service-activator method="processOrder" input-channel="syncChannel" ref="orderProcessor"/&gt;

&lt;int:service-activator method="processOrder" input-channel="asyncChannel" ref="orderProcessor"&gt;
    &lt;int:poller fixed-delay="2000"/&gt;
&lt;/int:service-activator&gt;</pre>

&nbsp;

The gateway is the entry point of the messaging system. It will receive the order and send it to the direct channel &#8220;requestChannel&#8221;. There, a router will redirect it to the appropriate channel based on the order id:

  * syncChannel: A <a href="http://docs.spring.io/spring-integration/docs/2.2.6.RELEASE/api/org/springframework/integration/channel/DirectChannel.html" target="_blank" rel="noopener">direct channel</a> that will send the order to an order processor subscribed to this channel.
  * asyncChannel: A <a href="http://docs.spring.io/spring-integration/docs/2.2.6.RELEASE/api/org/springframework/integration/channel/QueueChannel.html" target="_blank" rel="noopener">queue channel</a> from which the order processor will actively retrieve the order.

Once the order is processed, an order confirmation will be sent back to the gateway. Here is a graphic representing this:


<div>
  <a style="margin-left: 1em; margin-right: 1em;" href="http://2.bp.blogspot.com/--wYnf0qLMdw/VUZKrANXLhI/AAAAAAAAEnY/nulBW8-84l0/s1600/grafic.png"><img loading="lazy" class="alignnone" src="https://2.bp.blogspot.com/--wYnf0qLMdw/VUZKrANXLhI/AAAAAAAAEnY/nulBW8-84l0/s1600/grafic.png" alt="Integration flow about how error handling works in Spring Integration" width="378" height="400" border="0" /></a>
</div>

Ok, let&#8217;s start with the simplest case, synchronous sending using a Direct Channel.

## <span style="color: #0b5394;">2 Synchronous sending with Direct channel </span>

The order processor is subscribed to the &#8220;syncChannel&#8221; Direct Channel. The &#8220;processOrder&#8221; method will be invoked in the sender&#8217;s thread:

<pre class="lang:java decode:true ">public OrderConfirmation processOrder(Order order) {
    logger.info("Processing order {}", order.getId());
    
    if (isInvalidOrder(order)) {
        logger.info("Error while processing order [{}]", ERROR_INVALID_ID);
        throw new InvalidOrderException(ERROR_INVALID_ID);
    }
    
    return new OrderConfirmation("confirmed");
}</pre>

&nbsp;

Now, we will implement a test that will provoke an exception by sending an invalid order. This test will send an order to the gateway:

<pre class="lang:java decode:true ">public interface OrderService {
    @Gateway
    public OrderConfirmation sendOrder(Order order);
}</pre>

&nbsp;

#### <span style="color: #003366;">Implement the test</span>

<u>TestSyncErrorHandling.java</u>

<pre class="lang:java decode:true ">@ContextConfiguration(locations = {"/xpadro/spring/integration/config/int-config.xml"})
@RunWith(SpringJUnit4ClassRunner.class)
public class TestSyncErrorHandling {
    
    @Autowired
    private OrderService service;
    
    @Test
    public void testCorrectOrder() {
        OrderConfirmation confirmation = service.sendOrder(new Order(3, "a correct order"));
        Assert.assertNotNull(confirmation);
        Assert.assertEquals("confirmed", confirmation.getId());
    }
    
    @Test
    public void testSyncErrorHandling() {
        OrderConfirmation confirmation = null;
        try {
            confirmation = service.sendOrder(new Order(1, "an invalid order"));
            Assert.fail("Should throw a MessageHandlingException");
        } catch (MessageHandlingException e) {
            Assert.assertEquals(InvalidOrderException.class, e.getCause().getClass());
            Assert.assertNull(confirmation);
        }
    }
}</pre>
