---
id: 31
title: Spring Integration – Using RMI Channel Adapters
date: 2013-10-21T11:32:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/10/21/spring-integration-using-rmi-channel-adapters/
permalink: /2013/10/spring-integration-using-rmi-channel-adapters.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/10/spring-integration-using-rmi-channel.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/4976398074051326
categories:
  - Integration
  - Spring
tags:
  - Integration
  - RMI
  - Spring
---

This article explains how to send and receive messages over RMI using Spring Integration RMI channel adapters. It is composed of the following sections:

  * Implement the service: The first section focuses on creating and exposing a service.
  * Implement the client: Shows how to invoke the service using the MessagingTemplate class.
  * Abstracting SI logic: Finally, I’ve added another section explaining how to implement the same client abstracting all Spring Integration code. Hence, leaving the client focused on its business logic.

You can get the source code at <a href="https://github.com/xpadro/spring-integration/tree/master/rmi/rmi-basic" target="_blank" rel="noopener">my Github repository</a>.

## <span style="color: #0b5394;">1 Implementing the service</span>

This first part is pretty simple. The service is defined through annotations, so it will be auto detected by component scanning. It has a repository injected. It gets the data from an embedded database, as it will be shown in this same section:

<pre class="lang:java decode:true ">@Service("defaultEmployeeService")
public class EmployeeServiceImpl implements EmployeeService {
    @Autowired
    private EmployeeRepository employeeRepository;
    
    @Override
    public Employee retrieveEmployee(int id) {
        return employeeRepository.getEmployee(id);
    }
}</pre>

The repository is as follows:

<pre class="lang:java decode:true ">@Repository
public class EmployeeRepositoryImpl implements EmployeeRepository {
    private JdbcTemplate template;
    private RowMapper&lt;Employee&gt; rowMapper = new EmployeeRowMapper();
    private static final String SEARCH = "select * from employees where id = ?";
    private static final String COLUMN_ID = "id";
    private static final String COLUMN_NAME = "name";
    
    @Autowired
    public EmployeeRepositoryImpl(DataSource dataSource) {
        this.template = new JdbcTemplate(dataSource);
    }
    
    public Employee getEmployee(int id) {
        return template.queryForObject(SEARCH, rowMapper, id);
    }
    
    private class EmployeeRowMapper implements RowMapper&lt;Employee&gt; {
        public Employee mapRow(ResultSet rs, int i) throws SQLException {
            Employee employee = new Employee();
            employee.setId(rs.getInt(COLUMN_ID));
            employee.setName(rs.getString(COLUMN_NAME));
            
            return employee;
        }
    }
}</pre>

&nbsp;

## <span style="color: #003366;">2 Application configuration</span>

The following configuration exposes the service over RMI:

<pre class="lang:xhtml decode:true ">&lt;context:component-scan base-package="xpadro.spring.integration"/&gt;

&lt;int-rmi:inbound-gateway request-channel="requestEmployee"/&gt;

&lt;int:channel id="requestEmployee"/&gt;

&lt;int:service-activator method="retrieveEmployee" input-channel="requestEmployee" ref="defaultEmployeeService"/&gt;

&lt;bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager"&gt;
    &lt;property name="dataSource" ref="dataSource" /&gt;
&lt;/bean&gt;

&lt;!-- in-memory database --&gt;
&lt;jdbc:embedded-database id="dataSource"&gt;
    &lt;jdbc:script location="classpath:db/schemas/schema.sql" /&gt;
    &lt;jdbc:script location="classpath:db/schemas/data.sql" /&gt;
&lt;/jdbc:embedded-database&gt;</pre>

&nbsp;

Let’s focus on the lines with the ‘int’ namespace:

The <a href="http://www.eaipatterns.com/MessagingGateway.html" target="_blank" rel="noopener">gateway</a>’s function is to separate the plumbing from the messaging system from the rest of the application. In this way, it gets hidden from the business logic. A gateway is bi directional, so you have:

  * Inbound gateway: Brings a message into the application and waits for a response.
  * Outbound gateway: Invokes an external system and sends the response back to the application.

In this example, we are using a <a href="http://docs.spring.io/spring-integration/docs/2.2.6.RELEASE/reference/html/rmi.html" target="_blank" rel="noopener">RMI</a> inbound gateway. It will receive a message over RMI and send it to the requestEmployee channel, which is also defined here.

Finally, the <a href="http://www.eaipatterns.com/MessagingAdapter.html" target="_blank" rel="noopener">service activator</a> allows you to connect a spring bean to a message channel. Here, it is connected to the requestEmployee channel. The message will arrive to the channel and the service activator will invoke the retrieveEmployee method. Take into account that the ‘method’ attribute is not necessary if the bean has only one public method or has a method annotated with @ServiceActivator.

The response will then be sent to the reply channel. Since we didn’t define this channel, it will create a temporary reply channel.

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <p>
    <a style="margin-left: 1em; margin-right: 1em;" href="http://2.bp.blogspot.com/-zV1pdRZtcmA/VUZKM-CqKBI/AAAAAAAAEnA/Vl8prAT8LYM/s1600/temporaryChannel.png"><img loading="lazy" class="alignnone" src="https://2.bp.blogspot.com/-zV1pdRZtcmA/VUZKM-CqKBI/AAAAAAAAEnA/Vl8prAT8LYM/s1600/temporaryChannel.png" alt="temporary channel diagram" width="400" height="162" border="0" /></a>
  </p>
</div>

## <span style="color: #0b5394;">3 Implementing the client</span>

The client we are going to implement will invoke the service in order to retrieve an employee. To do this, it will use the <a href="http://docs.spring.io/spring-integration/docs/2.2.5.RELEASE/api/org/springframework/integration/core/MessagingTemplate.html" target="_blank" rel="noopener">MessagingTemplate</a> class:

<pre class="lang:java decode:true ">@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations={"classpath:xpadro/spring/integration/test/config/client-config.xml"})
public class TestRmiClient {
    @Autowired
    MessageChannel localChannel;
    
    @Autowired
    MessagingTemplate template;
    
    @Test
    public void retrieveExistingEmployee() {
        Employee employee = (Employee) template.convertSendAndReceive(localChannel, 2);
        
        Assert.assertNotNull(employee);
        Assert.assertEquals(2, employee.getId());
        Assert.assertEquals("Bruce Springsteen", employee.getName());
    }
}</pre>

The client uses the messagingTemplate to convert the Integer object to a Message and send it to the local channel. As shown below, there’s an outbound gateway connected to the local channel. This outbound gateway will send the request message over RMI.

<pre class="lang:xhtml decode:true ">&lt;int-rmi:outbound-gateway request-channel="localChannel" remote-channel="requestEmployee" host="localhost"/&gt;

&lt;int:channel id="localChannel"/&gt;

&lt;bean class="org.springframework.integration.core.MessagingTemplate" /&gt;</pre>

&nbsp;

## <span style="color: #0b5394;">4 Abstracting SI logic</span>

In the previous section, you may have noticed that the client class which accesses the service has Spring Integration specific logic mixed with its business code:

  * It uses the MessagingTemplate, which is a SI class.
  * It knows about the local channel, which is specific to the messaging system

In this section, I will implement this same example abstracting the messaging logic, so the client will only care about its business logic.

First, let’s take a look at the new client:

<pre class="lang:java decode:true ">@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations={"classpath:xpadro/spring/integration/test/config/client-gateway-config.xml"})
public class TestRmiGatewayClient {
    @Autowired
    private EmployeeService service;
    
    @Test
    public void retrieveExistingEmployee() {
        Employee employee = service.retrieveEmployee(2);
        
        Assert.assertNotNull(employee);
        Assert.assertEquals(2, employee.getId());
        Assert.assertEquals("Bruce Springsteen", employee.getName());
    }
}</pre>

We can see now that the client just implements its business logic, without using neither message channels nor messaging template. It will just call the service interface. All the messaging definitions are in the configuration file.

<u>client-gateway-config.xml</u>

<pre class="lang:xhtml decode:true ">&lt;int-rmi:outbound-gateway request-channel="localChannel" remote-channel="requestEmployee" host="localhost"/&gt;

&lt;int:channel id="localChannel"/&gt;

&lt;int:gateway default-request-channel="localChannel" 
    service-interface="xpadro.spring.integration.service.EmployeeService"/&gt;</pre>

What we did here is add a gateway that will intercept calls to the service interface EmployeeService. Spring Integration will use the <a href="http://docs.spring.io/spring-integration/api/org/springframework/integration/gateway/GatewayProxyFactoryBean.html" target="_blank" rel="noopener">GatewayProxyFactoryBean</a> class to create a proxy around the service interface. This proxy will use a messaging template to send the invocation to the request channel and wait for the response.

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://1.bp.blogspot.com/-RL4qwWR4zb4/VUZKM-9YcKI/AAAAAAAAEnE/SN0vs5nprxg/s1600/gatewayProxyFactoryBean.png"><img loading="lazy" class="alignnone" src="https://1.bp.blogspot.com/-RL4qwWR4zb4/VUZKM-9YcKI/AAAAAAAAEnE/SN0vs5nprxg/s1600/gatewayProxyFactoryBean.png" alt="Application flow using RMI Channel Adapters" width="400" height="231" border="0" /></a>
</div>

&nbsp;

## <span style="color: #0b5394;">5 Conclusion</span>

We have seen how to use Spring Integration to access a service over RMI.  We have also seen that we can not only explicitly send messages using the MessagingTemplate but also do it transparently with GatewayProxyFactoryBean.

&nbsp;