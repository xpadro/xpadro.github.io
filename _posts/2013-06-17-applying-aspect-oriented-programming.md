---
id: 35
title: Applying aspect oriented programming
date: 2013-06-17T07:42:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/06/17/applying-aspect-oriented-programming/
permalink: /2013/06/applying-aspect-oriented-programming.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/06/applying-aspect-oriented-programming.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/6346123645277474875
categories:
  - AOP
  - Spring
  - Spring MVC
tags:
  - AOP
  - MVC
  - REST
  - Spring
---

This article explains how to apply aspect oriented programming with Spring AOP. The main target of the aspect oriented programming is the separation of cross-cutting concerns. When we talk about cross-cutting concerns we are referring to generic functionality that is used in several places in our system or application.

These concepts are, among others:

  * Logging
  * Transaction management
  * Error handling
  * Monitoring
  * Security

&nbsp;

The way to achieve this separation is by modularizing these concepts. This will allow us to keep our business logic classes clean, containing only the code for which the class was designed. If we don&#8217;t modularize these concerns, it will lead to code tangling (the class contains different concerns) and code scattering (the same concern will be spread across the system).

In this example, we have a Spring MVC application that access the requested data (clients and orders) and shows a page with its information. We can take a look at the different layers:

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://1.bp.blogspot.com/-S48JaaCkRX0/VUZBWN2j0iI/AAAAAAAAEko/hJgHHr7wrcQ/s1600/aop1.png"><img loading="lazy" class="alignnone" src="http://1.bp.blogspot.com/-S48JaaCkRX0/VUZBWN2j0iI/AAAAAAAAEko/hJgHHr7wrcQ/s1600/aop1.png" alt="web application architecture diagram" width="640" height="454" border="0" /></a>
</div>

&nbsp;

In the above graphic, we can appreciate that there are functionalities spread across different classes (monitoring is implemented in every service), and some classes contain different concerns (for example, the class ClientController contains logging and exception handling). In order to fix that, we will write aspects to implement our cross-cutting concerns. The goal is to implement the following model:

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://4.bp.blogspot.com/-VuJNkAh2Ges/VUZBWNRZnrI/AAAAAAAAEkk/P1M_9mBsD30/s1600/aop2.png"><img loading="lazy" class="alignnone" src="http://4.bp.blogspot.com/-VuJNkAh2Ges/VUZBWNRZnrI/AAAAAAAAEkk/P1M_9mBsD30/s1600/aop2.png" alt="web application layers diagram" width="640" height="214" border="0" /></a>
</div>

&nbsp;

Each class contains only the business logic related code, while the aspects will be responsible of intercepting the code in order to inject the cross-cutting concerns.

Let&#8217;s see this with an example.

Source code can be found at <a href="https://github.com/xpadro/spring-aop" target="_blank" rel="noopener">my Github repository</a>.

## <span style="color: #0b5394;">1   Checking controller code</span>

ClientController:

<pre class="lang:java decode:true ">@Controller
public class ClientController {
    @Autowired
    private ClientService clientService;
    private static Logger mainLogger = LoggerFactory.getLogger("generic");
    private static Logger errorLogger = LoggerFactory.getLogger("errors");

    @RequestMapping("/getClients")
    public String getClients(Model model, @RequestParam("id") int id) {
        mainLogger.debug("Executing getClients request");
        
        try {
            Client client = clientService.getClient(id);
            model.addAttribute("client", client);
        } catch (DataAccessException e) {
            errorLogger.error("error in ClientController", e);
            NotificationUtils.sendNotification(e);
            return "errorPage";
        }
        
        return "showClient";
    }
}</pre>

The objective of this controller consists in retrieving a client and returning a view showing its information but, as you can see, this code contains additional logic. In one hand, it handles exceptions that may be thrown by the service and it redirects to an error page. On the other hand, it generates logging information and notification sending in case of error. All this code is generic to all controllers in this application (and probably to other classes).

It&#8217;s true that we could have used the @ControllerAdvice annotation to centralize exception handling, but the target of this post is to see how to accomplish it with Spring AOP.

The same happens with the order controller. I won&#8217;t include it here because I don&#8217;t want to make the post unnecessary long. If you want to check it out, you can get the source code included in the previous link.

## <span style="color: #0b5394;">2   Checking services code</span>

ClientService:

<pre class="lang:java decode:true ">@Service("clientService")
public class ClientServiceImpl implements ClientService {
    @Autowired
    private ClientRepository clientRepository;
    private static Logger mainLogger = LoggerFactory.getLogger("generic");
    private static Logger monitorLogger = LoggerFactory.getLogger("monitoring");
    
    @Override
    @Transactional(readOnly = true)
    public Client getClient(int id) {
        mainLogger.debug("Accessing client service");
        long startTime = System.currentTimeMillis();
        Client client = clientRepository.getClient(id);
        long totalTime = System.currentTimeMillis() - startTime;
        monitorLogger.info("Invocation time {}ms ", totalTime);
        
        return client;
    }
}</pre>

&nbsp;

In addition to the service invocation, it also contains logging generation and monitoring of the execution time in each invocation.

We could also use aspects to modularize transaction management if we needed to use programmatic transaction management, but it&#8217;s not the case in this example.

## <span style="color: #0b5394;">3   Data access layer</span>

ClientRepositoryImpl:

<pre class="lang:java decode:true ">@Repository
public class ClientRepositoryImpl implements ClientRepository {
    private JdbcTemplate template;
    private RowMapper&lt;Client&gt; rowMapper = new ClientRowMapper();
    private static final String SEARCH = "select * from clients where clientId = ?";
    private static final String COLUMN_ID = "clientId";
    private static final String COLUMN_NAME = "name";
    
    public ClientRepositoryImpl() {}
    
    public ClientRepositoryImpl(DataSource dataSource) {
        this.template = new JdbcTemplate(dataSource);
    }
    
    public Client getClient(int id) {
        return template.queryForObject(SEARCH, rowMapper, id);
    }
    
    private class ClientRowMapper implements RowMapper&lt;Client&gt; {
        public Client mapRow(ResultSet rs, int i) throws SQLException {
            Client client = new Client();
            client.setClientId(rs.getInt(COLUMN_ID));
            client.setName(rs.getString(COLUMN_NAME));
            
            return client;
        }
    }
}</pre>

&nbsp;

This code does not contain any cross-cutting concern but I&#8217;ve included it to show all the sample application layers.

## <span style="color: #0b5394;">4   Activating AOP</span>

To configure AOP, it is necessary to import the following dependencies:

<pre class="lang:xhtml decode:true ">&lt;dependency&gt;
    &lt;groupId&gt;org.springframework&lt;/groupId&gt;
    &lt;artifactId&gt;spring-aop&lt;/artifactId&gt;
    &lt;version&gt;3.2.1.RELEASE&lt;/version&gt;
&lt;/dependency&gt;
&lt;dependency&gt;
    &lt;groupId&gt;org.aspectj&lt;/groupId&gt;
    &lt;artifactId&gt;aspectjweaver&lt;/artifactId&gt;
    &lt;version&gt;1.6.8&lt;/version&gt;
&lt;/dependency&gt;</pre>

&nbsp;

In the Spring configuration file, we need to add the following tags:

<pre class="lang:xhtml decode:true ">&lt;context:component-scan base-package="xpadro.spring.mvc.aop"/&gt;
&lt;aop:aspectj-autoproxy/&gt;</pre>

The component-scan tag will search within the base package in order to find our aspects. To use auto scan, you not only need to define the aspect class with @Aspect annotation, but also you will need to include @Component annotation. If you don&#8217;t include @Component you will need to define the aspect in the xml configuration file.

## <span style="color: #0b5394;">5   Centralizing error handling</span>

We&#8217;ll write an aspect with an @Around advice. This advice will intercept every method annotated with @RequestMapping annotation and will be responsible of invoking it, catching exceptions thrown by the service:

<pre class="lang:java decode:true ">@Component
@Aspect
public class CentralExceptionHandler {
    private static Logger errorLogger = LoggerFactory.getLogger("errors");
    
    @Around("@annotation(org.springframework.web.bind.annotation.RequestMapping) && target(controller)")
    public String handleException(ProceedingJoinPoint jp, Object controller) throws Throwable {
        String view = null;
        
        try {
            view = (String) jp.proceed();
        } catch (DataAccessException e) {
            errorLogger.error("error in {}", controller.getClass().getSimpleName(), e);
            NotificationUtils.sendNotification(e);
            return "errorPage";
        }
        
        return view;
    }
}</pre>

&nbsp;

The @Target annotation allows us to reference the intercepted class. Now we have the exception handling handled by the aspect so we can get rid of this logic in our controllers:

<pre class="lang:java decode:true ">@Controller
public class ClientController {
    @Autowired
    private ClientService clientService;
    private static Logger mainLogger = LoggerFactory.getLogger("generic");
    //private static Logger errorLogger = LoggerFactory.getLogger("errors");
    
    @RequestMapping("/getClients")
    public String getClients(Model model, @RequestParam("id") int id) {
        mainLogger.debug("Executing getClients request");
        
        //try {
            Client client = clientService.getClient(id);
            model.addAttribute("client", client);
        //} catch (DataAccessException e) {
            //errorLogger.error("error in ClientController", e);
            //NotificationUtils.sendNotification(e);
            //return "errorPage";
        //}
        
        return "showClient";
    }	
}</pre>

Obviously, this is an example to see which lines were removed. Do not leave commented code! 😉

Just a note, you could have intercepted exceptions thrown by the controller with the following advice:

<pre class="lang:java decode:true ">@AfterThrowing(pointcut="@annotation(org.springframework.web.bind.annotation.RequestMapping)", throwing="e")</pre>

&nbsp;

But be aware that this advice will not prevent the exception from propagating.

## <span style="color: #0b5394;">6   Centralizing logging</span>

The logging aspect will have two advices, one for controller logging and another one for service logging:

<pre class="lang:java decode:true ">@Aspect
@Component
public class CentralLoggingHandler {
    private static Logger mainLogger = LoggerFactory.getLogger("generic");
    
    @Before("@annotation(org.springframework.web.bind.annotation.RequestMapping) && @annotation(mapping)")
    public void logControllerAccess(RequestMapping mapping) {
        mainLogger.debug("Executing {} request", mapping.value()[0]);
    }
    
    @Before("execution(* xpadro.spring.mvc.*..*Service+.*(..)) && target(service)")
    public void logServiceAccess(Object service) {
        mainLogger.debug("Accessing {}", service.getClass().getSimpleName());
    }
}</pre>

&nbsp;

## <span style="color: #0b5394;">7   Finally, the monitoring concern</span>

We will write another aspect for monitoring concern. The advice is as follows:

<pre class="lang:java decode:true">@Aspect
@Component
public class CentralMonitoringHandler {
    private static Logger monitorLogger = LoggerFactory.getLogger("monitoring");
    
    @Around("execution(* xpadro.spring.mvc.*..*Service+.*(..)) && target(service)")
    public Object logServiceAccess(ProceedingJoinPoint jp, Object service) throws Throwable {
        long startTime = System.currentTimeMillis();
        Object result = jp.proceed();
        long totalTime = System.currentTimeMillis() - startTime;
        monitorLogger.info("{}|Invocation time {}ms ", service.getClass().getSimpleName(), totalTime);
        
        return result;
    }
}</pre>

&nbsp;

## <span style="color: #0b5394;">8   Checking the final code</span>

After we have modularized all the cross-cutting concerns, our controllers and services contain only the business logic:

<pre class="lang:java decode:true">@Controller
public class ClientController {
    @Autowired
    private ClientService clientService;
    
    @RequestMapping("/getClients")
    public String getClients(Model model, @RequestParam("id") int id) {
    	Client client = clientService.getClient(id);
    	model.addAttribute("client", client);
    	
    	return "showClient";
    }	
}


@Service("clientService")
public class ClientServiceImpl implements ClientService {
    @Autowired
    private ClientRepository clientRepository;
    
    @Override
    @Transactional(readOnly = true)
    public Client getClient(int id) {
        return clientRepository.getClient(id);
    }
}</pre>

&nbsp;

## <span style="color: #0b5394;">9   Conclusion </span>

To sum up, we have seen how to apply aspect oriented programming to keep our code clean and focused to the logic for which it was designed. Before using AOP, just take into account its known limitations.

&nbsp;