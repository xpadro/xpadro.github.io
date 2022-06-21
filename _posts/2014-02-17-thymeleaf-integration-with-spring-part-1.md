---
id: 27
title: Thymeleaf integration with Spring (Part 1)
date: 2014-02-17T15:07:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2014/02/17/thymeleaf-integration-with-spring-part-1/
permalink: /2014/02/thymeleaf-integration-with-spring-part-1.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2014/02/thymeleaf-integration-with-spring-part-1.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/7221605335859443740
categories:
  - mongoDB
  - Spring
  - Spring MVC
  - Thymeleaf
tags:
  - MongoDB
  - MVC
  - Spring
  - Thymeleaf
---

This article is focused on how Thymeleaf can be integrated with the Spring framework. This will let our MVC web application take advantage of Thymeleaf HTML5 template engine without losing any of the Spring features. The data layer uses Spring Data to interact with a mongoDB database.

The example consists in a Hotel&#8217;s single page web application from where we can send two different requests:

  * Insert a new guest: A synchronous request that shows how Thymeleaf is integrated with Spring&#8217;s form backing beans.
  * List guests: An asynchronous request that shows how to handle fragment rendering with AJAX.

This tutorial expects you to know the basics of Thymeleaf. If not, you should first read <a href="http://www.thymeleaf.org/doc/html/Using-Thymeleaf.html" target="_blank" rel="noopener">this article</a>.

Here&#8217;s an example of the application flow:

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://4.bp.blogspot.com/-Escg7ZAil6Q/VUZZguKWzQI/AAAAAAAAEpc/spoeIGUy-24/s1600/hotelFlow.png"><img loading="lazy" class="alignnone" src="https://4.bp.blogspot.com/-Escg7ZAil6Q/VUZZguKWzQI/AAAAAAAAEpc/spoeIGUy-24/s1600/hotelFlow.png" alt="Thymeleaf integration with Spring - app flow diagram" width="320" height="316" border="0" /></a>
</div>

&nbsp;

This example is based on Thymeleaf <a href="http://www.thymeleaf.org/whatsnew21.html" target="_blank" rel="noopener">2.1</a> and Spring 4 versions.

The source code can be found at <a href="https://github.com/xpadro/thymeleaf/tree/master/th-spring-integration" target="_blank" rel="noopener">my Github repository</a>.

## <span style="color: #0b5394;">1 Configuration</span>

This tutorial takes the <a href="http://docs.spring.io/spring/docs/4.0.1.RELEASE/spring-framework-reference/htmlsingle/#beans-java" target="_blank" rel="noopener">JavaConfig </a>approach to configure the required beans. This means xml configuration files are no longer necessary.

#### <span style="color: #003366;">web.xml</span>

Since we want to use JavaConfig, we need to specify <a href="http://docs.spring.io/spring/docs/4.0.x/javadoc-api/org/springframework/web/context/support/AnnotationConfigWebApplicationContext.html" target="_blank" rel="noopener">AnnotationConfigWebApplicationContext </a>as the class that will configure the Spring container. If we don&#8217;t specify it, it will use <a href="http://docs.spring.io/spring/docs/4.0.x/javadoc-api/org/springframework/web/context/support/XmlWebApplicationContext.html" target="_blank" rel="noopener">XmlWebApplicationContext </a>by default.

When defining where the configuration files are located, we can specify classes or packages. Here, I&#8217;m indicating my configuration class.

<pre class="lang:xhtml decode:true ">&lt;!-- Bootstrap the root context --&gt;
&lt;listener&gt;
    &lt;listener-class&gt;org.springframework.web.context.ContextLoaderListener&lt;/listener-class&gt;
&lt;/listener&gt;

&lt;!-- Configure ContextLoaderListener to use AnnotationConfigWebApplicationContext --&gt;
&lt;context-param&gt;
    &lt;param-name&gt;contextClass&lt;/param-name&gt;
    &lt;param-value&gt;
        org.springframework.web.context.support.AnnotationConfigWebApplicationContext
    &lt;/param-value&gt;
&lt;/context-param&gt;

&lt;!-- @Configuration classes or package --&gt;
&lt;context-param&gt;
    &lt;param-name&gt;contextConfigLocation&lt;/param-name&gt;
    &lt;param-value&gt;xpadro.thymeleaf.configuration.WebAppConfiguration&lt;/param-value&gt;
&lt;/context-param&gt;

&lt;!-- Spring servlet --&gt;
&lt;servlet&gt;
    &lt;servlet-name&gt;springServlet&lt;/servlet-name&gt;
    &lt;servlet-class&gt;org.springframework.web.servlet.DispatcherServlet&lt;/servlet-class&gt;
    &lt;init-param&gt;
        &lt;param-name&gt;contextClass&lt;/param-name&gt;
        &lt;param-value&gt;org.springframework.web.context.support.AnnotationConfigWebApplicationContext&lt;/param-value&gt;
    &lt;/init-param&gt;
&lt;/servlet&gt;
&lt;servlet-mapping&gt;
    &lt;servlet-name&gt;springServlet&lt;/servlet-name&gt;
    &lt;url-pattern&gt;/spring/*&lt;/url-pattern&gt;
&lt;/servlet-mapping&gt;</pre>

&nbsp;

#### <span style="color: #003366;">Spring Configuration</span>

My configuration is split in two classes: thymeleaf-spring integration (<a href="https://github.com/xpadro/thymeleaf/blob/master/th-spring-integration/src/main/java/xpadro/thymeleaf/configuration/WebAppConfiguration.java" target="_blank" rel="noopener">WebAppConfiguration </a>class) and mongoDB configuration (<a href="https://github.com/xpadro/thymeleaf/blob/master/th-spring-integration/src/main/java/xpadro/thymeleaf/configuration/MongoDBConfiguration.java" target="_blank" rel="noopener">MongoDBConfiguration </a>class).

WebAppConfiguration.java

<pre class="lang:java decode:true ">@EnableWebMvc
@Configuration
@ComponentScan("xpadro.thymeleaf")
@Import(MongoDBConfiguration.class)
public class WebAppConfiguration extends WebMvcConfigurerAdapter {
    @Bean
    @Description("Thymeleaf template resolver serving HTML 5")
    public ServletContextTemplateResolver templateResolver() {
        ServletContextTemplateResolver templateResolver = new ServletContextTemplateResolver();
        templateResolver.setPrefix("/WEB-INF/html/");
        templateResolver.setSuffix(".html");
        templateResolver.setTemplateMode("HTML5");
        
        return templateResolver;
    }
    
    @Bean
    @Description("Thymeleaf template engine with Spring integration")
    public SpringTemplateEngine templateEngine() {
        SpringTemplateEngine templateEngine = new SpringTemplateEngine();
        templateEngine.setTemplateResolver(templateResolver());
        
        return templateEngine;
    }
    
    @Bean
    @Description("Thymeleaf view resolver")
    public ThymeleafViewResolver viewResolver() {
        ThymeleafViewResolver viewResolver = new ThymeleafViewResolver();
        viewResolver.setTemplateEngine(templateEngine());
        
        return viewResolver;
    }
    
    @Bean
    @Description("Spring message resolver")
    public ResourceBundleMessageSource messageSource() {  
        ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();  
        messageSource.setBasename("i18n/messages");  
        
        return messageSource;  
    }
    
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**").addResourceLocations("/WEB-INF/resources/");
    }
}</pre>

&nbsp;

Things to highlight from looking at the above code:

  * <a href="http://docs.spring.io/spring/docs/4.0.1.RELEASE/javadoc-api/org/springframework/web/servlet/config/annotation/EnableWebMvc.html" target="_blank" rel="noopener">@EnableWebMvc</a>: This enables Spring MVC annotations like @RequestMapping. This would be the same as the xml namespace <mvc:annotation-driven />
  * <a href="http://docs.spring.io/spring/docs/4.0.1.RELEASE/javadoc-api/org/springframework/context/annotation/ComponentScan.html" target="_blank" rel="noopener">@ComponentScan(“xpadro.thymeleaf”)</a>: Activates component scanning in the xpadro.thymeleaf package and subpackages. Classes annotated with @Component and related annotations will be registered as beans.
  * We are registering three beans which are necessary to configure Thymeleaf and integrate it with the Spring framework. 
      * template resolver: Resolves template names and delegates them to a servlet context resource resolver.
      * template engine: Integrates with Spring framework, establishing the Spring specific dialect as the default dialect.
      * view resolver: Thymeleaf implementation of the Spring MVC view resolver interface in order to resolve Thymeleaf views.

&nbsp;

MongoDBConfiguration.java

<pre class="lang:java decode:true ">@Configuration
@EnableMongoRepositories("xpadro.thymeleaf.repository")
public class MongoDBConfiguration extends AbstractMongoConfiguration {
    @Override
    protected String getDatabaseName() {
        return "hotel-db";
    }
    
    @Override
    public Mongo mongo() throws Exception {
        return new Mongo();
    }
}</pre>

&nbsp;

This class extends <a href="http://docs.spring.io/spring-data/data-mongodb/docs/1.3.3.RELEASE/api/org/springframework/data/mongodb/config/AbstractMongoConfiguration.html" target="_blank" rel="noopener">AbstracMongoConfiguration</a>, which defines mongoFactory and mongoTemplate beans.  
The @EnableMongoRepositories will scan the specified package in order to find interfaces extending MongoRepository. Then, it will create a bean for each one. We will see this later, at the data access layer section.

## <span style="color: #0b5394;">2 Thymeleaf – Spring MVC Integration</span>

#### <span style="color: #003366;">HotelController</span>

The controller is responsible for accessing the service layer, construct the view model from the result and return a view. With the configuration that we set in the previous section, now MVC Controllers will be able to return a view Id that will be resolved as a Thymeleaf view.

Below we can see a fragment of the controller where it handles the initial request (http://localhost:8080/th-spring-integration/spring/home):

<pre class="lang:java decode:true ">@Controller
public class HotelController {
    @Autowired
    private HotelService hotelService;
    
    @ModelAttribute("guest")
    public Guest prepareGuestModel() {
        return new Guest();
    }
    
    @ModelAttribute("hotelData")
    public HotelData prepareHotelDataModel() {
        return hotelService.getHotelData();
    }
    
    @RequestMapping(value = "/home", method = RequestMethod.GET)
    public String showHome(Model model) {
        prepareHotelDataModel();
        prepareGuestModel();
        
        return "home";
    }
    
    ...
}</pre>

&nbsp;

A typical MVC Controller that returns a &#8220;home&#8221; view id. Thymeleaf template resolver will look for a template named &#8220;home.html&#8221; which is located in /WEB-INF/html/ folder, as indicated in the configuration. Additionally, a view attribute named &#8220;hotelData&#8221; will be exposed to the Thymeleaf view, containing hotel information that needs to be displayed on the initial view.

This fragment of the home view shows how it accesses some of the properties of the view attribute by using Spring Expression Language (Spring EL):

<pre class="lang:xhtml decode:true ">&lt;span th:text="${hotelData.name}"&gt;Hotel name&lt;/span&gt;&lt;br /&gt;
&lt;span th:text="${hotelData.address}"&gt;Hotel address&lt;/span&gt;&lt;br /&gt;</pre>

&nbsp;

Another nice feature is that Thymeleaf will be able to resolve Spring managed message properties, which have been configured through the MessageSource interface.

<pre class="lang:xhtml decode:true ">&lt;h3 th:text="#{hotel.information}"&gt;Hotel Information&lt;/h3&gt;</pre>

&nbsp;

#### <span style="color: #003366;">Error handling</span>

Trying to add a new user will raise an exception if a user with the same id already exists. The exception will be handled and the home view will be rendered with an error message.

Since we only have one controller, there&#8217;s no need to use <a href="http://xpadro.blogspot.com/2013/03/centralize-validation-and-exception.html" target="_blank" rel="noopener">@ControllerAdvice</a>. We will instead use a @ExceptionHandler annotated method. You can notice that we are returning an internationalized message as the error message:

<pre class="lang:java decode:true ">@ExceptionHandler({GuestFoundException.class})
public ModelAndView handleDatabaseError(GuestFoundException e) {
    ModelAndView modelAndView = new ModelAndView();
    modelAndView.setViewName("home");
    modelAndView.addObject("errorMessage", "error.user.exist");
    modelAndView.addObject("guest", prepareGuestModel());
    modelAndView.addObject("hotelData", prepareHotelDataModel());
    
    return modelAndView;
}</pre>

&nbsp;

Thymeleaf will resolve the view attribute with ${} and then it will resolve the message #{}:

<pre class="lang:xhtml decode:true ">&lt;span class="messageContainer" th:unless="${#strings.isEmpty(errorMessage)}" th:text="#{${errorMessage}}"&gt;&lt;/span&gt;</pre>

&nbsp;

The th:unless Thymeleaf attribute will only render the span element if an error message has been returned.

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://4.bp.blogspot.com/-qpic4j9TmL0/VUZZgigHtPI/AAAAAAAAEpY/TLNl_tPEkGk/s1600/hotelError.png"><img loading="lazy" class="alignnone" src="https://4.bp.blogspot.com/-qpic4j9TmL0/VUZZgigHtPI/AAAAAAAAEpY/TLNl_tPEkGk/s1600/hotelError.png" alt="Thymeleaf integration with Spring - Part 1. View with error message" width="640" height="158" border="0" /></a>
</div>

&nbsp;

## <span style="color: #0b5394;">3 The Service layer</span>

The service layer accesses the data access layer and adds some business logic.

<pre class="lang:java decode:true ">@Service("hotelServiceImpl")
public class HotelServiceImpl implements HotelService {
    @Autowired
    HotelRepository hotelRepository;
    
    @Override
    public List&lt;Guest&gt; getGuestsList() {
        return hotelRepository.findAll();
    }
    
    @Override
    public List&lt;Guest&gt; getGuestsList(String surname) {
        return hotelRepository.findGuestsBySurname(surname);
    }
    
    @Override
    public void insertNewGuest(Guest newGuest) {
        if (hotelRepository.exists(newGuest.getId())) {
            throw new GuestFoundException();
        }
        
        hotelRepository.save(newGuest);
    }
}</pre>

&nbsp;

## <span style="color: #0b5394;">4 The Data Access layer</span>

The HotelRepository extends the Spring Data class <a href="http://docs.spring.io/spring-data/mongodb/docs/1.3.3.RELEASE/api/org/springframework/data/mongodb/repository/MongoRepository.html" target="_blank" rel="noopener">MongoRepository</a>.

<pre class="lang:java decode:true ">public interface HotelRepository extends MongoRepository&lt;Guest, Long&gt; {
    @Query("{ 'surname' : ?0 }")
    List&lt;Guest&gt; findGuestsBySurname(String surname);
}</pre>

&nbsp;

This is just an interface, we won&#8217;t implement it. If you remember the configuration class, we added the following annotation:

<pre class="lang:java decode:true ">@EnableMongoRepositories("xpadro.thymeleaf.repository")</pre>

&nbsp;

Since this is the package where the repository is located, Spring will create a bean and inject a mongoTemplate to it. Extending this interface provides us with generic CRUD operations. If you need additional operations, you can add them with the @Query annotation (see code above).

&nbsp;

## <span style="color: #0b5394;">5 Conclusion</span>

We have configured Thymeleaf to resolve views in a Spring managed web application. This allows the view to access to Spring Expression Language and message resolving. The next part of this tutorial is going to show how forms are linked to Spring form backing beans and how we can reload fragments by sending an AJAX request.

Read the next part of this tutorial: <a href="http://xpadro.com/2014/02/thymeleaf-integration-with-spring-part-2.html" target="_blank" rel="noopener">Thymeleaf integration with Spring (Part 2)</a>

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.

&nbsp;