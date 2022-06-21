---
id: 43
title: Create and test REST services with Spring MVC
date: 2013-02-05T13:12:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/02/05/create-and-test-rest-services-with-spring-mvc/
permalink: /2013/02/create-and-test-rest-services-with-spring-mvc.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/02/create-and-test-rest-services-with.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/6073335687786225174
categories:
  - Spring
  - Spring MVC
tags:
  - MVC
  - REST
  - Spring
  - Test
---

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;">The first part of this example shows how to create and test REST services using Spring MVC. The controller contains CRUD operations on warehouses and its products. For this example, the repository is a stub that simulates access to the database.</span>
</div>

<div style="margin-bottom: 0cm;">
</div>

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;">The second part will access these services using the RestTemplate class and test them.</span>
</div>

<div style="margin-bottom: 0cm;">
</div>

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;">Source code available at the <a href="https://github.com/xpadro/spring-rest/tree/master/rest_test" target="_blank" rel="noopener">github spring rest repository</a>.</span>
</div>

<div style="margin-bottom: 0cm;">
</div>

<h2 style="margin-bottom: 0cm;">
  <span style="color: #073763;"><span style="mso-ansi-language: ES;">Configuration</span></span>
</h2>

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;"> The context configuration is quite simple. It is split in two xml files. The parent context:</span>
</div>

<div style="margin-bottom: 0cm;">
</div>

<div style="line-height: normal; margin-bottom: 0cm; mso-layout-grid-align: none; text-autospace: none;">
  <pre class="lang:xhtml decode:true ">&lt;?xml version="1.0" encoding="UTF-8"?&gt;
&lt;beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
             http://www.springframework.org/schema/beans
             http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
             http://www.springframework.org/schema/context
             http://www.springframework.org/schema/context/spring-context-3.0.xsd
             http://www.springframework.org/schema/mvc
             http://www.springframework.org/schema/mvc/spring-mvc-3.0.xsd"&gt;
 
       &lt;!-- Detects annotations like @Component, @Service, @Controller... --&gt;
       &lt;context:component-scan base-package="xpadro.tutorial.rest"/&gt;
      
       &lt;!-- Detects MVC annotations like @RequestMapping --&gt;
       &lt;mvc:annotation-driven/&gt;
&lt;/beans&gt;</pre>
  
  <p>
    &nbsp;
  </p>
</div>

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;">And the servlet context, which contains the stub repository:</span>
</div>

<div style="margin-bottom: 0cm;">
</div>

<div style="line-height: normal; margin-bottom: 0cm; mso-layout-grid-align: none; text-autospace: none;">
  <pre class="lang:xhtml decode:true ">&lt;?xml version="1.0" encoding="UTF-8"?&gt;
&lt;beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
             http://www.springframework.org/schema/beans
             http://www.springframework.org/schema/beans/spring-beans-3.0.xsd"&gt;
      
       &lt;!-- The warehouse repository. Simulates the retrieval of data from the database --&gt;
       &lt;bean id="warehouseRepository" class="xpadro.tutorial.rest.repository.WarehouseRepositoryImpl"/&gt;
&lt;/beans&gt;</pre>
  
  <p>
    &nbsp;
  </p>
</div>

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;">The web.xml file just contains basic Spring configuration:</span>
</div>

<div style="margin-bottom: 0cm;">
</div>

<div style="line-height: normal; margin-bottom: 0cm; mso-layout-grid-align: none; text-autospace: none;">
  <pre class="lang:xhtml decode:true ">&lt;?xml version="1.0" encoding="UTF-8"?&gt;
&lt;web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"xmlns="http://java.sun.com/xml/ns/javaee" xmlns:web="http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd" id="WebApp_ID" version="2.5"&gt;
  &lt;display-name&gt;SpringRestTest&lt;/display-name&gt;
 
  &lt;!-- Root context configuration --&gt;
  &lt;context-param&gt;
       &lt;param-name&gt;contextConfigLocation&lt;/param-name&gt;
       &lt;param-value&gt;classpath:xpadro/tutorial/rest/configuration/root-context.xml&lt;/param-value&gt;
  &lt;/context-param&gt;
 
  &lt;!-- Loads Spring root context, which will be the parent context --&gt;
  &lt;listener&gt;
       &lt;listener-class&gt;org.springframework.web.context.ContextLoaderListener&lt;/listener-class&gt;
  &lt;/listener&gt;
 
  &lt;!-- Spring servlet--&gt;
  &lt;servlet&gt;
       &lt;servlet-name&gt;springServlet&lt;/servlet-name&gt;
       &lt;servlet-class&gt;org.springframework.web.servlet.DispatcherServlet&lt;/servlet-class&gt;
       &lt;init-param&gt;
             &lt;param-name&gt;contextConfigLocation&lt;/param-name&gt;
             &lt;param-value&gt;classpath:xpadro/tutorial/rest/configuration/app-context.xml&lt;/param-value&gt;
       &lt;/init-param&gt;
  &lt;/servlet&gt;
  &lt;servlet-mapping&gt;
       &lt;servlet-name&gt;springServlet&lt;/servlet-name&gt;
       &lt;url-pattern&gt;/spring/*&lt;/url-pattern&gt;
  &lt;/servlet-mapping&gt;
&lt;/web-app&gt;</pre>
  
  <p>
    &nbsp;
  </p>
</div>

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;">And finally the pom.xml with all the dependencies, which can be found <a href="https://github.com/xpadro/spring-rest/blob/master/rest_test/pom.xml" target="_blank" rel="noopener">in the repository pom.xml file</a>.</span>
</div>

<div style="margin-bottom: 0cm;">
</div>

<h2 style="margin-bottom: 0cm;">
  <span style="color: #073763;"><span style="mso-ansi-language: ES;">Creating the RESTful services</span></span>
</h2>

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;"> The controller has the following methods:</span>
</div>

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;"> </span>
</div>

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;"><b>getWarehouse</b>: Returns an existing warehouse. </span>
</div>

<div style="margin-bottom: 0cm;">
</div>

<div style="line-height: normal; margin-bottom: 0cm; mso-layout-grid-align: none; text-autospace: none;">
  <pre class="lang:java decode:true ">@RequestMapping(value="/warehouses/{warehouseId}", method=RequestMethod.GET)
public @ResponseBody Warehouse getWarehouse(@PathVariable("warehouseId") int id) {
     return warehouseRepository.getWarehouse(id);
}</pre>
  
  <p>
    &nbsp;
  </p>
</div>

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;">This method uses several MVC annotations, explained below:</span>
</div>

  * <span style="mso-ansi-language: ES;">@RequestMapping: This annotation maps requests based on method onto specific handlers, in this case, the getWarehouse method, but only if the HTTP request method is GET. Specifying the method, you can have multiple methods mapped to the same uri. For example, the following request will be handled by this method and return the warehouse identified by 1:</span>

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;"><span style="mso-tab-count: 1;">                </span>http://localhost:8080/myApp/spring/warehouses/1</span>
</div>

&nbsp;

  * <span style="mso-ansi-language: ES;">@PathVariable: Extract values from request URL. In the method above, it extracts the warehouseId value from the request URL and maps it to the id parameter.</span>

  * <span style="mso-ansi-language: ES;">@ResponseBody: Bounds the return value of the method to the response body. For this task it uses HTTP message converters. The function of these converters is to convert between HTTP request/response and object.</span>

&nbsp;

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;"><b>addProduct</b>: Adds a new product to an existing warehouse.</span>
</div>

<div style="margin-bottom: 0cm;">
</div>

<div style="line-height: normal; margin-bottom: 0cm; mso-layout-grid-align: none; text-autospace: none;">
  <pre class="lang:java decode:true ">@RequestMapping(value="/warehouses/{warehouseId}/products", method=RequestMethod.POST)
@ResponseStatus(HttpStatus.CREATED)
public void addProduct(@PathVariable("warehouseId") int warehouseId, @RequestBody Product product, HttpServletRequest request, HttpServletResponse response) {
            
     warehouseRepository.addProduct(warehouseId, product);
     response.setHeader("Location", request.getRequestURL().append("/")
          .append(product.getId()).toString());
}</pre>
  
  <p>
    &nbsp;
  </p>
</div>

  * <span style="mso-ansi-language: ES;">With the @ResponseStatus annotation, we are defining that there won’t be a view returned. Instead, we will return a response with an empty body.</span>

  * <span style="mso-ansi-language: ES;">Like @ResponseBody annotation, the @RequestBody annotation uses converters to transform request data into the object passed as a parameter.</span>

&nbsp;

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;"> Other methods are defined in this controller but won’t put them all here. You can look up the source code linked above.</span>
</div>

<div style="margin-bottom: 0cm;">
</div>

<div style="margin-bottom: 0cm;">
</div>

<h2 style="margin-bottom: 0cm;">
  <span style="color: #073763;"><span style="mso-ansi-language: ES;">Setting the exception handler</span></span>
</h2>

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;"> You can have multiple exception handlers, each one mapped to one or more exception types. Using the @ExceptionHandler annotation allows you to handle exceptions raised by methods annotated with @RequestMapping. Instead of forwarding to a view, it allows you to set a response status code. For example:</span>
</div>

<div style="margin-bottom: 0cm;">
</div>

<div style="line-height: normal; margin-bottom: 0cm; mso-layout-grid-align: none; text-autospace: none;">
  <pre class="lang:java decode:true ">@ResponseStatus(HttpStatus.NOT_FOUND)
@ExceptionHandler({ProductNotFoundException.class})
public voidhandleProductNotFound(ProductNotFoundException pe) {
     logger.warn("Product not found. Code: "+pe.getMessage());
}</pre>
  
  <p>
    &nbsp;
  </p>
</div>

<h2 style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;"> </span><span style="color: #073763;"><span style="mso-ansi-language: ES;">Testing the services</span></span>
</h2>

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;"> The test class is as follows:</span>
</div>

<div style="margin-bottom: 0cm;">
  <span style="mso-ansi-language: ES;">  </span>
</div>

<div style="line-height: normal; margin-bottom: 0cm; mso-layout-grid-align: none; text-autospace: none;">
  <pre class="lang:java decode:true ">@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations={
       "classpath:xpadro/tutorial/rest/configuration/root-context.xml",
       "classpath:xpadro/tutorial/rest/configuration/app-context.xml"})
public class WarehouseTesting {
     private static final int WAREHOUSE_ID = 1;
     private static final int PRODUCT_ID = 4;
      
     private RestTemplate restTemplate = new RestTemplate();

     /**
      * Tests accessing to an existing warehouse
      */
     @Test
     public void getWarehouse() {
          String uri = "http://localhost:8081/rest_test/spring/warehouses/{warehouseId}";
          Warehouse warehouse = restTemplate.getForObject(uri, Warehouse.class, WAREHOUSE_ID);
          assertNotNull(warehouse);
          assertEquals("WAR_BCN_004", warehouse.getName());
     }
      
     /**
      * Tests the addition of a new product to an existing warehouse.
      */
     @Test
     public void addProduct() {
          //Adds the new product
          String uri = "http://localhost:8081/rest_test/spring/warehouses/{warehouseId}/products";
          Product product = new Product(PRODUCT_ID, "PROD_999");
          URI newProductLocation = restTemplate.postForLocation(uri, product, WAREHOUSE_ID);
            
          //Checks we can access to the created product
          Product createdProduct = restTemplate.getForObject(newProductLocation, Product.class);
          assertEquals(product, createdProduct);
          assertNotNull(createdProduct.getId());
     }

     /**
      * Tests the removal of an existing product
      */
     @Test
     public void removeProduct() {
          String uri = "http://localhost:8081/rest_test/spring/warehouses/{warehouseId}/products/{productId}";
          restTemplate.delete(uri, WAREHOUSE_ID, PRODUCT_ID);
            
          try {
               restTemplate.getForObject(uri, Product.class, WAREHOUSE_ID, PRODUCT_ID);
               throw new AssertionError("Should have returned an 404 error code");
          } catch (HttpClientErrorException e) {
               assertEquals(HttpStatus.NOT_FOUND, e.getStatusCode());
          }
     }
}</pre>
</div>

&nbsp;

## Related posts

<li style="margin-bottom: 0cm;">
  <a href="http://xpadro.com/2013/02/20/accessing-restful-services-http-message-converters/">Accessing Restful services. HTTP Message converters</a>
</li>

<div style="margin-bottom: 0cm;">
</div>