---
id: 28
title: Migrating Spring MVC RESTful web services to Spring 4
date: 2014-01-20T07:23:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2014/01/20/migrating-spring-mvc-restful-web-services-to-spring-4/
permalink: /2014/01/migrating-spring-mvc-restful-web-services-to-spring-4.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2014/01/migrating-spring-mvc-restful-web.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/5169342842346804974
categories:
  - mongoDB
  - Spring
  - Spring MVC
tags:
  - MVC
  - REST
  - Spring
  - Test
  - web services
---

Spring 4 brings several <a href="http://docs.spring.io/spring/docs/4.0.0.RELEASE/spring-framework-reference/htmlsingle/#_general_web_improvements" target="_blank" rel="noopener">improvements</a> for MVC applications. In this post I will focus on restful web services and try these improvements by performing a migration from Spring MVC 3.2 to Spring 4.0. We will take a project implemented with Spring 3.2 and perform the steps to upgrade it to Spring 4.0. The following points sum up the content of this post:

  * Migration from Spring 3.2 to Spring 4.0
  * Changes in <a href="http://docs.spring.io/spring/docs/4.0.0.RELEASE/javadoc-api/org/springframework/web/bind/annotation/ResponseBody.html" target="_blank" rel="noopener">@ResponseBody</a> and inclusion of <a href="http://docs.spring.io/spring/docs/4.0.0.RELEASE/javadoc-api/org/springframework/web/bind/annotation/RestController.html" target="_blank" rel="noopener">@RestController</a>
  * Synchronous and Asynchronous calls

&nbsp;

The source code of the following projects can be found at github:

<a href="https://github.com/xpadro/spring-rest/tree/master/spring-rest-api-v32" target="_blank" rel="noopener">Original project (spring 3.2)</a>

<a href="https://github.com/xpadro/spring-rest/tree/master/spring-rest-api-v4" target="_blank" rel="noopener">Migration to Spring 4</a>

&nbsp;

## <span style="color: #0b5394;">1 The Spring 3.2 RESTful sample</span>

The starting project is implemented with Spring 3.2 ([pom.xml](https://github.com/xpadro/spring-rest/blob/master/spring-rest-api-v32/pom.xml)) . It consists in a Spring MVC application that access a database to retrieve data about TV series. Let&#8217;s have a look at its REST API to see it clearer:

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://1.bp.blogspot.com/-1JqVa9KNu_8/VUZZOApzqQI/AAAAAAAAEpE/AHqEHrROr6Q/s1600/blog-rest-api.png"><img loading="lazy" class="alignnone" src="http://1.bp.blogspot.com/-1JqVa9KNu_8/VUZZOApzqQI/AAAAAAAAEpE/AHqEHrROr6Q/s1600/blog-rest-api.png" alt="migration from Spring MVC 3.2 to Spring 4.0 - REST api diagram" width="320" height="232" border="0" /></a>
</div>

&nbsp;

<span style="color: #003366;"><b>Spring configuration</b></span>

root-context.xml

<pre class="lang:xhtml decode:true ">&lt;import resource="db-context.xml"/&gt;

&lt;!-- Detects annotations like @Component, @Service, @Controller, @Repository, @Configuration --&gt;
&lt;context:component-scan base-package="xpadro.spring.web.controller,xpadro.spring.web.service"/&gt;

&lt;!-- Detects MVC annotations like @RequestMapping --&gt;
&lt;mvc:annotation-driven/&gt;</pre>

&nbsp;

db-context.xml

<pre class="lang:xhtml decode:true ">&lt;!-- Registers a mongo instance --&gt;
&lt;bean id="mongo" class="org.springframework.data.mongodb.core.MongoFactoryBean"&gt;
    &lt;property name="host" value="localhost" /&gt;
&lt;/bean&gt;

&lt;bean id="mongoTemplate" class="org.springframework.data.mongodb.core.MongoTemplate"&gt;
    &lt;constructor-arg name="mongo" ref="mongo" /&gt;
    &lt;constructor-arg name="databaseName" value="rest-db" /&gt;
&lt;/bean&gt;</pre>

&nbsp;

#### <span style="color: #003366;"><b>Service implementation</b></span>

This class is responsible of retrieving the data from a mongoDB database:

<pre class="lang:java decode:true">@Service
public class SeriesServiceImpl implements SeriesService {
    
    @Autowired
    private MongoOperations mongoOps;
    
    @Override
    public Series[] getAllSeries() {
        List&lt;Series&gt; seriesList = mongoOps.findAll(Series.class);
        return seriesList.toArray(new Series[0]);
    }
    
    @Override
    public Series getSeries(long id) {
        return mongoOps.findById(id, Series.class);
    }
    
    @Override
    public void insertSeries(Series series) {
        mongoOps.insert(series);
    }
    
    @Override
    public void deleteSeries(long id) {
        Query query = new Query();
        Criteria criteria = new Criteria("_id").is(id);
        query.addCriteria(criteria);
        
        mongoOps.remove(query, Series.class);
    }
}</pre>

&nbsp;

#### <span style="color: #003366;"><b>Controller implementation</b></span>

This controller will handle requests and interact with the service in order to retrieve series data:

<pre class="lang:java decode:true ">@Controller
@RequestMapping(value="/series")
public class SeriesController {
    
    private SeriesService seriesService;
    
    @Autowired
    public SeriesController(SeriesService seriesService) {
        this.seriesService = seriesService;
    }
    
    @RequestMapping(method=RequestMethod.GET)
    @ResponseBody
    public Series[] getAllSeries() {
        return seriesService.getAllSeries();
    }
    
    @RequestMapping(value="/{seriesId}", method=RequestMethod.GET)
    public ResponseEntity&lt;Series&gt; getSeries(@PathVariable("seriesId") long id) {
        Series series = seriesService.getSeries(id);
        
        if (series == null) {
            return new ResponseEntity&lt;Series&gt;(HttpStatus.NOT_FOUND);
        }
        
        return new ResponseEntity&lt;Series&gt;(series, HttpStatus.OK);
    }
    
    @RequestMapping(method=RequestMethod.POST)
    @ResponseStatus(HttpStatus.CREATED)
    public void insertSeries(@RequestBody Series series, HttpServletRequest request, HttpServletResponse response) {
        seriesService.insertSeries(series);
        response.setHeader("Location", request.getRequestURL().append("/").append(series.getId()).toString());
    }
    
    @RequestMapping(value="/{seriesId}", method=RequestMethod.DELETE)
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteSeries(@PathVariable("seriesId") long id) {
        seriesService.deleteSeries(id);
    }
}</pre>

&nbsp;

#### <span style="color: #003366;"><b>Integration testing</b></span>

These integration tests will test our controller within a mock Spring MVC environment. In this way, we will be able to also test the mappings of our handler methods. For this purpose, the [MockMvc](http://docs.spring.io/spring-framework/docs/3.2.x/javadoc-api/org/springframework/test/web/servlet/MockMvc.html) class becomes very useful. If you want to learn how to write tests of Spring MVC controllers I highly recommend the [Spring MVC Test Tutorial](http://www.petrikainulainen.net/spring-mvc-test-tutorial/) series by Petri Kainulainen.

<pre class="lang:java decode:true ">@RunWith(SpringJUnit4ClassRunner.class)
@WebAppConfiguration
@ContextConfiguration(locations={
    "classpath:xpadro/spring/web/test/configuration/test-root-context.xml",
    "classpath:xpadro/spring/web/configuration/app-context.xml"})
public class SeriesIntegrationTest {
    private static final String BASE_URI = "/series";
    
    private MockMvc mockMvc;
    
    @Autowired
    private WebApplicationContext webApplicationContext;
    
    @Autowired
    private SeriesService seriesService;
    
    @Before
    public void setUp() {
        reset(seriesService);
        mockMvc = MockMvcBuilders.webAppContextSetup(webApplicationContext).build();
        
        when(seriesService.getAllSeries()).thenReturn(new Series[]{
            new Series(1, "The walking dead", "USA", "Thriller"), 
            new Series(2, "Homeland", "USA", "Drama")});
            
        when(seriesService.getSeries(1L)).thenReturn(new Series(1, "Fringe", "USA", "Thriller"));
    }
    
    @Test
    public void getAllSeries() throws Exception {
        mockMvc.perform(get(BASE_URI)
            .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(content().contentType("application/json;charset=UTF-8"))
            .andExpect(jsonPath("$", hasSize(2)))
            .andExpect(jsonPath("$[0].id", is(1)))
            .andExpect(jsonPath("$[0].name", is("The walking dead")))
            .andExpect(jsonPath("$[0].country", is("USA")))
            .andExpect(jsonPath("$[0].genre", is("Thriller")))
            .andExpect(jsonPath("$[1].id", is(2)))
            .andExpect(jsonPath("$[1].name", is("Homeland")))
            .andExpect(jsonPath("$[1].country", is("USA")))
            .andExpect(jsonPath("$[1].genre", is("Drama")));
            
        verify(seriesService, times(1)).getAllSeries();
        verifyZeroInteractions(seriesService);
    }
    
    @Test
    public void getJsonSeries() throws Exception {
        mockMvc.perform(get(BASE_URI + "/{seriesId}", 1L)
            .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(content().contentType("application/json;charset=UTF-8"))
            .andExpect(jsonPath("$.id", is(1)))
            .andExpect(jsonPath("$.name", is("Fringe")))
            .andExpect(jsonPath("$.country", is("USA")))
            .andExpect(jsonPath("$.genre", is("Thriller")));
        
        verify(seriesService, times(1)).getSeries(1L);
        verifyZeroInteractions(seriesService);
    }
    
    @Test
    public void getXmlSeries() throws Exception {
        mockMvc.perform(get(BASE_URI + "/{seriesId}", 1L)
            .accept(MediaType.APPLICATION_XML))
            .andExpect(status().isOk())
            .andExpect(content().contentType(MediaType.APPLICATION_XML))
            .andExpect(xpath("/series/id").string("1"))
            .andExpect(xpath("/series/name").string("Fringe"))
            .andExpect(xpath("/series/country").string("USA"))
            .andExpect(xpath("/series/genre").string("Thriller"));
            
        verify(seriesService, times(1)).getSeries(1L);
        verifyZeroInteractions(seriesService);
    }
}</pre>

I&#8217;m showing some of the tests implemented. Check [SeriesIntegrationTesting](https://github.com/xpadro/spring-rest/blob/master/spring-rest-api-v32/src/test/java/xpadro/spring/web/test/SeriesIntegrationTesting.java) for complete implementation.

&nbsp;

<span style="color: #003366;"><b>Functional testing</b></span>

The application contains some functional testing by using the [RestTemplate](http://docs.spring.io/spring/docs/3.2.x/javadoc-api/org/springframework/web/client/RestTemplate.html) class. You need the webapp deployed in order to test this.

<pre class="lang:java decode:true ">@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations={
    "classpath:xpadro/spring/web/configuration/root-context.xml",
    "classpath:xpadro/spring/web/configuration/app-context.xml"})
public class SeriesFunctionalTesting {
    private static final String BASE_URI = "http://localhost:8080/spring-rest-api-v32/spring/series";
    private RestTemplate restTemplate = new RestTemplate();
    
    @Autowired
    private MongoOperations mongoOps;
    
    @Before
    public void setup() {
        List&lt;HttpMessageConverter&lt;?&gt;&gt; converters = new ArrayList&lt;HttpMessageConverter&lt;?&gt;&gt;();
        converters.add(new StringHttpMessageConverter());
        converters.add(new Jaxb2RootElementHttpMessageConverter());
        converters.add(new MappingJacksonHttpMessageConverter());
        restTemplate.setMessageConverters(converters);
        
        initializeDatabase();
    }
    
    private void initializeDatabase() {
        try {
            mongoOps.dropCollection("series");
            
            mongoOps.insert(new Series(1, "The walking dead", "USA", "Thriller"));
            mongoOps.insert(new Series(2, "Homeland", "USA", "Drama"));
        } catch (DataAccessResourceFailureException e) {
            fail("MongoDB instance is not running");
        }
    }
    
    @Test
    public void getAllSeries() {
        Series[] series = restTemplate.getForObject(BASE_URI, Series[].class);
        
        assertNotNull(series);
        assertEquals(2, series.length);
        assertEquals(1L, series[0].getId());
        assertEquals("The walking dead", series[0].getName());
        assertEquals("USA", series[0].getCountry());
        assertEquals("Thriller", series[0].getGenre());
        assertEquals(2L, series[1].getId());
        assertEquals("Homeland", series[1].getName());
        assertEquals("USA", series[1].getCountry());
        assertEquals("Drama", series[1].getGenre());
    }
    
    @Test
    public void getJsonSeries() {
        List&lt;HttpMessageConverter&lt;?&gt;&gt; converters = new ArrayList&lt;HttpMessageConverter&lt;?&gt;&gt;();
        converters.add(new MappingJacksonHttpMessageConverter());
        restTemplate.setMessageConverters(converters);
        
        String uri = BASE_URI + "/{seriesId}";
        ResponseEntity&lt;Series&gt; seriesEntity = restTemplate.getForEntity(uri, Series.class, 1l);
        assertNotNull(seriesEntity.getBody());
        assertEquals(1l, seriesEntity.getBody().getId());
        assertEquals("The walking dead", seriesEntity.getBody().getName());
        assertEquals("USA", seriesEntity.getBody().getCountry());
        assertEquals("Thriller", seriesEntity.getBody().getGenre());
        assertEquals(MediaType.parseMediaType("application/json;charset=UTF-8"), seriesEntity.getHeaders().getContentType());
    }
    
    @Test
    public void getXmlSeries() {
        String uri = BASE_URI + "/{seriesId}";
        ResponseEntity&lt;Series&gt; seriesEntity = restTemplate.getForEntity(uri, Series.class, 1L);
        assertNotNull(seriesEntity.getBody());
        assertEquals(1l, seriesEntity.getBody().getId());
        assertEquals("The walking dead", seriesEntity.getBody().getName());
        assertEquals("USA", seriesEntity.getBody().getCountry());
        assertEquals("Thriller", seriesEntity.getBody().getGenre());
        assertEquals(MediaType.APPLICATION_XML, seriesEntity.getHeaders().getContentType());
    }
}</pre>

&nbsp;

That&#8217;s all, the web application is tested and running. Now is time to migrate to Spring 4.

&nbsp;

## <span style="color: #0b5394;">2 Migrating to Spring 4</span>

Check [this page](https://github.com/spring-projects/spring-framework/wiki/Migrating-from-earlier-versions-of-the-spring-framework) to read information about migrating from earlier versions of the Spring framework

### <span style="color: #0b5394;">2.1 Changing maven dependencies</span>

This section explains which dependencies should be modified. You can take a look at the complete pom.xml [here](https://github.com/xpadro/spring-rest/blob/master/spring-rest-api-v4/pom.xml).

The first step is to change Spring dependencies version from 3.2.3.RELEASE to 4.0.0.RELEASE:

<pre class="lang:xhtml decode:true ">&lt;dependency&gt;
    &lt;groupId&gt;org.springframework&lt;/groupId&gt;
    &lt;artifactId&gt;spring-context&lt;/artifactId&gt;
    &lt;version&gt;4.0.0.RELEASE&lt;/version&gt;
&lt;/dependency&gt;

&lt;dependency&gt;
    &lt;groupId&gt;org.springframework&lt;/groupId&gt;
    &lt;artifactId&gt;spring-webmvc&lt;/artifactId&gt;
    &lt;version&gt;4.0.0.RELEASE&lt;/version&gt;
&lt;/dependency&gt;</pre>

&nbsp;

The next step is to update to Servlet 3.0 specification. This step is important since some of the Spring features are based on Servlet 3.0 and won&#8217;t be available. In fact, trying to execute SeriesIntegrationTesting will result in a ClassNotFoundException due to this reason, which is also explained [this Spring JIRA ticket](https://jira.springsource.org/browse/SPR-11049).

<pre class="lang:xhtml decode:true ">&lt;dependency&gt;
    &lt;groupId&gt;javax.servlet&lt;/groupId&gt;
    &lt;artifactId&gt;javax.servlet-api&lt;/artifactId&gt;
    &lt;version&gt;3.1.0&lt;/version&gt;
&lt;/dependency&gt;</pre>

&nbsp;

### <span style="color: #0b5394;">2.2 Updating of Spring namespace</span>

Don&#8217;t forget to change the namespace of your spring configuration files:

<pre class="lang:default decode:true ">http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd
http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd
</pre>

Review the information page linked in section 2 since there are some changes regarding mvc namespace.

### <span style="color: #0b5394;">2.3 Deprecation of jackson libraries</span>

If you check SeriesFunctionalTesting (setup method) again you will notice that the Jackson converter is now deprecated. If you try to run the test it will throw a NoSuchMethodError due to method change in Jackson libraries:

<span style="font-size: x-small;">java.lang.NoSuchMethodError: org.codehaus.jackson.map.ObjectMapper.getTypeFactory()Lorg/codehaus/jackson/map/type/TypeFactory</span>

In Spring 4, support to Jackson 1.x has been deprecated in favor of Jackson v2. Let&#8217;s change the old dependency:

<pre class="lang:xhtml decode:true ">&lt;dependency&gt;
    &lt;groupId&gt;org.codehaus.jackson&lt;/groupId&gt;
    &lt;artifactId&gt;jackson-mapper-asl&lt;/artifactId&gt;
    &lt;version&gt;1.4.2&lt;/version&gt;
&lt;/dependency&gt;</pre>

For these:

<pre class="lang:xhtml decode:true ">&lt;dependency&gt;
    &lt;groupId&gt;com.fasterxml.jackson.core&lt;/groupId&gt;
    &lt;artifactId&gt;jackson-core&lt;/artifactId&gt;
    &lt;version&gt;2.3.0&lt;/version&gt;
&lt;/dependency&gt;

&lt;dependency&gt;
    &lt;groupId&gt;com.fasterxml.jackson.core&lt;/groupId&gt;
    &lt;artifactId&gt;jackson-databind&lt;/artifactId&gt;
    &lt;version&gt;2.3.0&lt;/version&gt;
&lt;/dependency&gt;</pre>

Finally, if you are explicitly registering message converters you will need to change the deprecated class for the new version:

<pre class="lang:java decode:true">//converters.add(new MappingJacksonHttpMessageConverter());
converters.add(new MappingJackson2HttpMessageConverter());</pre>

&nbsp;

### <span style="color: #0b5394;">2.4 Migration complete</span>

The migration is done. Now you can run the application and execute its tests. The next section will review some of the improvements I mentioned at the beginning of this post.

&nbsp;

## <span style="color: #0b5394;">3 Spring 4 Web improvements </span>

### <span style="color: #0b5394;">3.1 @ResponseBody and @RestController </span>

If your REST API serves content in JSON or XML format, some of the API methods (annotated with @RequestMapping) will have its return type annotated with @ResponseBody. With this annotation present, the return type will be included into the response body. In Spring 4 we can simplify this in two ways:

**Annotate the controller with @ResponseBody**  
This annotation can now be added on type level. In this way, the annotation is inherited and we are not forced to put this annotation in every method.

<pre class="lang:java decode:true ">@Controller
@ResponseBody
public class SeriesController {</pre>

&nbsp;

**Annotate the controller with @RestController**

<pre class="lang:java decode:true ">@RestController
public class SeriesController {</pre>

&nbsp;

This annotation simplifies the controller even more. If we check this annotation we will see that it is itself annotated with @Controller and @ResponseBody:

<pre class="lang:java decode:true ">@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller
@ResponseBody
public @interface RestController {</pre>

Including this annotation won&#8217;t affect methods annotated with @ResponseEntity. The handler adapter looks up into a list of return value handlers in order to resolve who is capable of handling the response. The [handler](http://docs.spring.io/spring/docs/4.0.0.RELEASE/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/HttpEntityMethodProcessor.html) responsible of handling the ResponseEntity return type is asked before the ResponseBody type, so it will be used if ResponseEntity annotation is present at the method.

### <span style="color: #0b5394;">3.2 Asynchronous calls </span>

Using the utility class RestTemplate for calling a RESTful service will block the thread until it receives a response. Spring 4 includes [AsyncRestTemplate](http://docs.spring.io/spring/docs/4.0.0.RELEASE/spring-framework-reference/htmlsingle/#rest-async-resttemplate) in order to execute asynchronous calls. Now you can make the call, continue doing other calculations and retrieve the response later.

<pre class="lang:java decode:true ">@Test
public void getAllSeriesAsync() throws InterruptedException, ExecutionException {
    logger.info("Calling async /series");
    Future&lt;ResponseEntity&lt;Series[]&gt;&gt; futureEntity = asyncRestTemplate.getForEntity(BASE_URI, Series[].class);
    logger.info("Doing other async stuff...");
    
    logger.info("Blocking to receive response...");
    ResponseEntity&lt;Series[]&gt; entity = futureEntity.get();
    logger.info("Response received");
    Series[] series = entity.getBody();
    
    assertNotNull(series);
    assertEquals(2, series.length);
    assertEquals(1L, series[0].getId());
    assertEquals("The walking dead", series[0].getName());
    assertEquals("USA", series[0].getCountry());
    assertEquals("Thriller", series[0].getGenre());
    assertEquals(2L, series[1].getId());
    assertEquals("Homeland", series[1].getName());
    assertEquals("USA", series[1].getCountry());
    assertEquals("Drama", series[1].getGenre());
}</pre>

&nbsp;

**Asynchronous calls with callback**  
Although the previous example makes an asynchronous call, the thread will block if we try to retrieve the response with futureEntity.get() if the response hasn&#8217;t already been sent.  
AsyncRestTemplate returns [ListenableFuture](http://docs.spring.io/spring/docs/4.0.0.RELEASE/javadoc-api/org/springframework/util/concurrent/ListenableFuture.html), which extends Future and allows us to register a callback. The following example makes an asynchronous call and keeps going with its own tasks. When the service returns a response, it will be handled by the callback:

<pre class="lang:java decode:true ">@Test
public void getAllSeriesAsyncCallable() throws InterruptedException, ExecutionException {
    logger.info("Calling async callable /series");
    ListenableFuture&lt;ResponseEntity&lt;Series[]&gt;&gt; futureEntity = asyncRestTemplate.getForEntity(BASE_URI, Series[].class);
    futureEntity.addCallback(new ListenableFutureCallback&lt;ResponseEntity&lt;Series[]&gt;&gt;() {
        @Override
        public void onSuccess(ResponseEntity&lt;Series[]&gt; entity) {
            logger.info("Response received (async callable)");
            Series[] series = entity.getBody();
            validateList(series);
        }
        
        @Override
        public void onFailure(Throwable t) {
            fail();
        }
    });
    
    logger.info("Doing other async callable stuff ...");
    Thread.sleep(6000); //waits for the service to send the response
}</pre>

&nbsp;

## <span style="color: #0b5394;">4 Conclusion </span>

We took a Spring 3.2.x web application and migrated it to the new release of Spring 4.0.0. We also reviewed some of the improvements that can be applied to a Spring 4 web application.

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.

&nbsp;