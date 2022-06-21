---
id: 13
title: Understanding Callable and Spring DeferredResult
date: 2015-07-13T11:52:00+01:00
author: xpadro
layout: post
permalink: /2015/07/understanding-callable-and-spring-deferredresult.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2015/07/understanding-callable-and-spring.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/4226607967432529477
categories:
  - Spring
  - Spring MVC
  - Spring-Boot
tags:
  - MVC
  - REST
  - Spring
  - Spring Boot
---

Asynchronous support introduced in Servlet 3.0 offers the possibility to process an HTTP request in another thread. This is specially interesting when you have a long running task, since while another thread processes this request, the container thread is freed and can continue serving other requests.

This topic has been explained many times, but there seems to be a little bit of confusion regarding those classes provided by the Spring framework which take advantage of this functionality. I am talking about returning Callable and Spring DeferredResult from a @Controller.

In this post I will implement both examples in order to show its differences.

All the examples shown here consist on implementing a controller which will execute a long running task, and then return the result to the client. The long running task is processed by the TaskService:

<pre class="lang:java decode:true ">@Service
public class TaskServiceImpl implements TaskService {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());
    
    @Override
    public String execute() {
        try {
            Thread.sleep(5000);
            logger.info("Slow task executed");
            return "Task finished";
        } catch (InterruptedException e) {
            throw new RuntimeException();
        }
    }
}</pre>

&nbsp;

The web application is built with <a href="http://projects.spring.io/spring-boot/" target="_blank" rel="noopener">Spring Boot</a>. We will be executing the following class to run our examples:

<pre class="lang:java decode:true ">@SpringBootApplication
public class MainApp {
    
    public static void main(String[] args) {
        SpringApplication.run(MainApp.class, args);
    }
}</pre>

&nbsp;

The source code with all these examples can be found at the Github <a href="https://github.com/xpadro/spring-rest/tree/master/async-processing" target="_blank" rel="noopener">Spring-Rest repository</a>.

## <span style="color: #003366;">1 Starting with a blocking controller</span>

In this example, a request arrives to the controller. The servlet thread won&#8217;t be released until the long running method is executed and we exit the @RequestMapping annotated method.

<pre class="lang:java decode:true ">@RestController
public class BlockingController {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());
    private final TaskService taskService;
    
    @Autowired
    public BlockingController(TaskService taskService) {
        this.taskService = taskService;
    }
    
    @RequestMapping(value = "/block", method = RequestMethod.GET, produces = "text/html")
    public String executeSlowTask() {
        logger.info("Request received");
        String result = taskService.execute();
        logger.info("Servlet thread released");
        
        return result;
    }
}</pre>

&nbsp;

If we run this example at http://localhost:8080/block, looking at the logs, we can see that the servlet request is not released until the long running task has been processed (5 seconds later):

<span style="font-size: x-small;">2015-07-12 12:41:11.849  [nio-8080-exec-6] x.s.web.controller.BlockingController    : Request received</span>  
<span style="font-size: x-small;">2015-07-12 12:41:16.851  [nio-8080-exec-6] x.spring.web.service.TaskServiceImpl     : Slow task executed</span>  
<span style="font-size: x-small;">2015-07-12 12:41:16.851  [nio-8080-exec-6] x.s.web.controller.BlockingController    : Servlet thread released</span>

## <span style="color: #003366;">2 Returning Callable</span>

In this example, instead of returning directly the result, we will return a Callable:

<pre class="lang:java decode:true ">@RestController
public class AsyncCallableController {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());
    private final TaskService taskService;
    
    @Autowired
    public AsyncCallableController(TaskService taskService) {
        this.taskService = taskService;
    }
    
    @RequestMapping(value = "/callable", method = RequestMethod.GET, produces = "text/html")
    public Callable&lt;String&gt; executeSlowTask() {
        logger.info("Request received");
        Callable&lt;String&gt; callable = taskService::execute;
        logger.info("Servlet thread released");
        
        return callable;
    }
}</pre>

&nbsp;

Returning Callable implies that Spring MVC will invoke the task defined in the Callable in a different thread. Spring will manage this thread by using a TaskExecutor. Before waiting for the long task to finish, the servlet thread will be released.

Let&#8217;s take a look at the logs:

<span style="font-size: x-small;">2015-07-12 13:07:07.012  [nio-8080-exec-5] x.s.w.c.AsyncCallableController          : Request received</span>  
<span style="font-size: x-small;">2015-07-12 13:07:07.013  [nio-8080-exec-5] x.s.w.c.AsyncCallableController          : Servlet thread released</span>  
<span style="font-size: x-small;">2015-07-12 13:07:12.014  [      MvcAsync2] x.spring.web.service.TaskServiceImpl     : Slow task executed</span>

You can see that we have returned from the servlet before the long running task has finished executing. This doesn&#8217;t mean the client has received a response. The communication with the client is still open waiting for the result, but the thread that received the request has been released and can serve another client&#8217;s request.

## <span style="color: #003366;">3 Returning DeferredResult</span>

First, we need to create a DeferredResult object. This object will be returned by the controller. What we will accomplish is the same with Callable, to release the servlet thread while we process the long running task in another thread.

<pre class="lang:java decode:true ">@RestController
public class AsyncDeferredController {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());
    private final TaskService taskService;
    
    @Autowired
    public AsyncDeferredController(TaskService taskService) {
        this.taskService = taskService;
    }
    
    @RequestMapping(value = "/deferred", method = RequestMethod.GET, produces = "text/html")
    public DeferredResult&lt;String&gt; executeSlowTask() {
        logger.info("Request received");
        DeferredResult&lt;String&gt; deferredResult = new DeferredResult&lt;&gt;();
        CompletableFuture.supplyAsync(taskService::execute)
            .whenCompleteAsync((result, throwable) -&gt; deferredResult.setResult(result));
        logger.info("Servlet thread released");
        
        return deferredResult;
    }
}</pre>

&nbsp;

So, what&#8217;s the difference from Callable? The difference is this time the thread is managed by us. It is our responsibility to set the result of the DeferredResult in a different thread.

What we have done in this example, is to create an asynchronous task with CompletableFuture. This will create a new thread where our long running task will be executed. Is in this thread where we will set the result.

From which pool are we retrieving this new thread? By default, the supplyAsync method in CompletableFuture will run the task in the ForkJoin pool. If you want to use a different thread pool, you can pass an executor to the supplyAsync method:

<pre class="lang:java decode:true ">public static &lt;U&gt; CompletableFuture&lt;U&gt; supplyAsync(Supplier&lt;U&gt; supplier, Executor executor)</pre>

&nbsp;

If we run this example, we will get the same result as with Callable:

<span style="font-size: x-small;">2015-07-12 13:28:08.433  [io-8080-exec-10] x.s.w.c.AsyncDeferredController          : Request received</span>  
<span style="font-size: x-small;">2015-07-12 13:28:08.475  [io-8080-exec-10] x.s.w.c.AsyncDeferredController          : Servlet thread released</span>  
<span style="font-size: x-small;">2015-07-12 13:28:13.469  [onPool-worker-1] x.spring.web.service.TaskServiceImpl     : Slow task executed </span>

## <span style="color: #003366;">4 Conclusion</span>

At a high level view, Callable and DeferredResult do the same exact thing, which is releasing the container thread and processing the long running task asynchronously in another thread. The difference is in who manages the thread executing the task.

Finally, if you want to learn more about threading and concurrency, please check my <a href="http://xpadro.com/2014/09/01/java-concurrency-tutorial/" target="_blank" rel="noopener">Java Concurrency tutorial</a>

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.

&nbsp;