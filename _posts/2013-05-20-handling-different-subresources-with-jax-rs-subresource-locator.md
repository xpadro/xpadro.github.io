---
id: 36
title: Handling different subresources with JAX-RS subresource locator
date: 2013-05-20T18:39:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/05/20/handling-different-subresources-with-jax-rs-subresource-locator/
permalink: /2013/05/handling-different-subresources-with-jax-rs-subresource-locator.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/05/handling-different-subresources-with.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/6023484590429549387
categories:
  - Java
  - REST
tags:
  - Jackson
  - JAX-RS
  - REST
---

In this article I won&#8217;t explain what a resource or sub resource is. There are several pages that explain perfectly well its meaning. For example, you can check <a href="http://docs.oracle.com/javaee/6/tutorial/doc/gknav.html" target="_blank" rel="noopener">Oracle tutorial</a> or <a href="https://jersey.java.net/nonav/documentation/2.0-m04/jaxrs-resources.html" target="_blank" rel="noopener">Jersey documentation</a>. I will focus on implementing a RESTful service with a JAX-RS sub resource locator. This locator will decide at runtime what type of sub resource will be returned.

You can check the source code on <a href="https://github.com/xpadro/spring-rest/tree/master/rest-subresources" target="_blank" rel="noopener">my Github repository</a>.

## <span style="color: #0b5394;">1 Configuration</span>

First of all, you need to include the JAX-RS reference implementation libraries:

<pre class="lang:xhtml decode:true ">&lt;properties&gt;
    &lt;jersey.version&gt;1.17.1&lt;/jersey.version&gt;
&lt;/properties&gt;

&lt;dependencies&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;com.sun.jersey&lt;/groupId&gt;
        &lt;artifactId&gt;jersey-server&lt;/artifactId&gt;
        &lt;version&gt;${jersey.version}&lt;/version&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;com.sun.jersey&lt;/groupId&gt;
        &lt;artifactId&gt;jersey-servlet&lt;/artifactId&gt;
        &lt;version&gt;${jersey.version}&lt;/version&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;com.sun.jersey&lt;/groupId&gt;
        &lt;artifactId&gt;jersey-json&lt;/artifactId&gt;
        &lt;version&gt;${jersey.version}&lt;/version&gt;
    &lt;/dependency&gt;
&lt;/dependencies&gt;</pre>

The first two dependencies are necessary if you want to develop services, while the third contains the implementation to convert your classes to JSON.

Next step is to define the Jersey servlet that will handle requests to our services. So, include the content below on your web.xml file:

<pre class="lang:xhtml decode:true ">&lt;servlet&gt;
    &lt;servlet-name&gt;Jersey Servlet&lt;/servlet-name&gt;
    &lt;servlet-class&gt;com.sun.jersey.spi.container.servlet.ServletContainer&lt;/servlet-class&gt;
    &lt;init-param&gt;
        &lt;param-name&gt;com.sun.jersey.config.property.packages&lt;/param-name&gt;
        &lt;param-value&gt;xpadro.rest.ri.resources&lt;/param-value&gt;
    &lt;/init-param&gt;
    &lt;init-param&gt;
        &lt;param-name&gt;com.sun.jersey.api.json.POJOMappingFeature&lt;/param-name&gt;
        &lt;param-value&gt;true&lt;/param-value&gt;
    &lt;/init-param&gt;
    &lt;load-on-startup&gt;1&lt;/load-on-startup&gt;
&lt;/servlet&gt;

&lt;servlet-mapping&gt;
    &lt;servlet-name&gt;Jersey Servlet&lt;/servlet-name&gt;
    &lt;url-pattern&gt;/rest/*&lt;/url-pattern&gt;
&lt;/servlet-mapping&gt;</pre>

&nbsp;

I have included two init parameters:

  * **com.sun.jersey.config.property.packages**: Required. It must define one or more packages separated by &#8220;;&#8221;. These are the packages where you put your resource classes.
  * **com.sun.jersey.api.json.POJOMappingFeature**: Activates POJO support, which means that it will use the Jackson library to convert your Java Objects to JSON and the other way back.

We are done with the configuration. Let&#8217;s implement the resources.

## <span style="color: #0b5394;">2 The root resource</span>

To declare a root resource you must annotate your class with @Path:

<pre class="lang:java decode:true ">import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.PathParam;

/**
 * Root resource
 */
@Path("/warehouse")
public class WarehouseResource {
    
    @GET
    public String getWarehouseInfo() {
        return "Warehouse location: Barcelona";
    }
    
    @Path("/items/{itemId}")
    public ItemResource getItem(@PathParam("itemId") Integer itemId) {
        ItemResource itemResource = null;
        
        if (itemId &gt; 10) {
            itemResource = new TypeAResource(itemId);
        }
        else {
            itemResource = new TypeBResource(itemId);
        }
        
        return itemResource;
    }
}</pre>

&nbsp;

The @GET annotated _getWarehouseInfo()_ method will handle requests to the root resource. So, when the user enters the following URI:

<div style="padding-left: 40px;">
  http://localhost:8080/rest-subresources/rest/warehouse
</div>

It will return the warehouse information.

Take into account that if I had included the @Path annotation in conjunction with @GET, I would be declaring a sub resource method.

The _getItem_ method is annotated with @Path but not with any request method designator (get, post&#8230;). This is because I&#8217;m declaring a sub resource locator. This sub resource locator will return an object  that will handle the HTTP request, but which one? It will depend on the id parameter. Both the _TypeAResource_ and _TypeBResource_ implement the same interface _ItemResource. S_o, we can return any of them.

## <span style="color: #0b5394;">3 Subresources</span>

These sub resources have a method with a request method designator, but no @Path annotation has been included:

<pre class="lang:java decode:true ">import javax.ws.rs.GET;
import javax.ws.rs.Produces;
import javax.ws.rs.WebApplicationException;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;
import xpadro.rest.ri.model.TypeAItem;
import xpadro.rest.ri.repository.TypeAItemRepository;

/**
 * Type A Subresource
 */
public class TypeAResource implements ItemResource {
    private int itemId;
    private TypeAItemRepository itemRepository = new TypeAItemRepository();
    
    public TypeAResource(int id) {
        this.itemId = id;
    }
    
    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public TypeAItem getTypeAItem() {
        TypeAItem item = itemRepository.retrieveItem(itemId);
        if (item == null) {
            throw new WebApplicationException(Response.Status.NOT_FOUND);
        }
        
        return item;
    }
}


import javax.ws.rs.GET;
import javax.ws.rs.Produces;
import javax.ws.rs.WebApplicationException;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;
import xpadro.rest.ri.model.TypeBItem;
import xpadro.rest.ri.repository.TypeBItemRepository;

/**
 * Type B Subresource
 */
public class TypeBResource implements ItemResource {
    private int itemId;
    private TypeBItemRepository itemRepository = new TypeBItemRepository();
    
    public TypeBResource(int id) {
        this.itemId = id;
    }
    
    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public TypeBItem getTypeBItem() {
        TypeBItem item = itemRepository.retrieveItem(itemId);
        if (item == null) {
            throw new WebApplicationException(Response.Status.NOT_FOUND);
        }
        
        return item;
    }
}</pre>

The following requests will return a different sub resource:

<div style="padding-left: 40px;">
  <p>
    http://localhost:8080/rest-subresources/rest/warehouse/items/1: Will return a Type B resource:
  </p>
  
  <p>
    {&#8220;id&#8221;:1,&#8221;price&#8221;:33.5,&#8221;type&#8221;:&#8221;B&#8221;}
  </p>
  
  <p>
    http://localhost:8080/rest-subresources/rest/warehouse/items/12: Will return a Type A resource:
  </p>
  
  <p>
    {&#8220;id&#8221;:12,&#8221;price&#8221;:35.5,&#8221;type&#8221;:&#8221;A&#8221;,&#8221;code&#8221;:&#8221;12SS34&#8243;}
  </p>
</div>

&nbsp;

## <span style="color: #0b5394;">4 Possible mistakes</span>

I&#8217;m including some errors that may occur if you fail to configure the application properly:

&nbsp;

### <span style="color: #003366;"><b>Fail on deploy</b>. com.sun.jersey.api.container.ContainerException</span>

The ResourceConfig instance does not contain any root resource classes

You did not define the packages init-param, so it will fail to find nor load your resource classes.

<pre class="lang:xhtml decode:true">&lt;init-param&gt;
    &lt;param-name&gt;com.sun.jersey.config.property.packages&lt;/param-name&gt;
    &lt;param-value&gt;xpadro.rest.ri.resources&lt;/param-value&gt;
&lt;/init-param&gt;</pre>

&nbsp;

### <span style="color: #003366;"><b>Fail on runtime</b>. com.sun.jersey.api.MessageException</span>

A message body writer for Java class [yourModelClass], and Java type class [yourModelClass], and MIME media type application/json was not found.

You did not define the JSON feature on web.xml. It won&#8217;t be able to convert your object to JSON:

<pre class="lang:xhtml decode:true ">&lt;init-param&gt;
    &lt;param-name&gt;com.sun.jersey.api.json.POJOMappingFeature&lt;/param-name&gt;
    &lt;param-value&gt;true&lt;/param-value&gt;
&lt;/init-param&gt;</pre>

&nbsp;

### <span style="color: #003366;"><b>Fail on runtime</b>. com.sun.jersey.api.MessageException</span>

A message body writer for Java class [yourModelClass], and Java type class [yourModelClass], and MIME media type application/json was not found.

This error might also be produced because the JSON maven dependency was not included:

<pre class="lang:xhtml decode:true ">&lt;dependency&gt;
    &lt;groupId&gt;com.sun.jersey&lt;/groupId&gt;
    &lt;artifactId&gt;jersey-json&lt;/artifactId&gt;
    &lt;version&gt;${jersey.version}&lt;/version&gt;
&lt;/dependency&gt;</pre>

&nbsp;

### <span style="color: #003366;"><b>Fail on deploy</b>. java.lang.ClassNotFoundException: com.sun.jersey.spi.container.servlet.ServletContainer</span>

The jersey-servlet maven dependency was not included. Older versions of Jersey included this class in jersey-core library, but in newer versions they have put it in a separate jar.

<pre class="lang:xhtml decode:true ">&lt;dependency&gt;
    &lt;groupId&gt;com.sun.jersey&lt;/groupId&gt;
    &lt;artifactId&gt;jersey-servlet&lt;/artifactId&gt;
    &lt;version&gt;${jersey.version}&lt;/version&gt;
&lt;/dependency&gt;</pre>

&nbsp;