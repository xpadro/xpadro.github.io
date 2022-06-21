---
id: 46
title: Configure Spring Webflow 2 with jsp views
description: Configuration of JSP views using Spring Webflow 2 framework
date: 2012-12-04T15:35:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2012/12/04/configure-webflow-2-with-jsp-views/
permalink: /2012/12/configure-webflow-2-with-jsp-views.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2012/12/configure-webflow-2-with-jsp-views.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/4845739814095014770
categories:
  - Spring
  - Spring Webflow
tags:
  - Spring
  - Webflow
---
<span style="font-family: Calibri;">This tutorial explains how to configure Spring Webflow 2, showing how it integrates with the view layer, in this case, JSP pages. The web application will allow two different types of request, flow executions and requests to Spring Web MVC. When a request comes in, it will try to find a flow. If not successful, it will try to map the request to an MVC handler.</span>

<div style="margin: 0cm 0cm 0pt;">
</div>

<h2 style="margin: 0cm 0cm 0pt;">
  <span style="color: #073763;">Environment</span>
</h2>

<div style="margin: 0cm 0cm 0pt;">
  <span style="font-family: Calibri;">JDK 1.6.0_24</span>
</div>

<span style="font-family: Calibri;">Spring 3.0.5.RELEASE</span>  
<span style="font-family: Calibri;">Webflow 2.3.1.RELEASE<span style="mso-spacerun: yes;">  </span></span>  
<span style="font-family: Calibri;">Tomcat 6.0</span>  
<span style="font-family: Calibri;">Maven 2</span>

<div style="margin: 0cm 0cm 0pt;">
</div>

## <span style="color: #073763;">Getting all needed files</span>

First of all you need to import the Spring libraries to your project. Using Maven 2, you just need to add the following dependencies:<span style="color: black; font-family: 'Courier New'; font-size: 10pt;"> </span>

<div style="line-height: normal; margin: 0cm 0cm 0pt; mso-layout-grid-align: none;">
  <pre class="lang:default decode:true EnlighterJSRAW">&lt;properties&gt;
      &lt;org.spring.version&gt;3.0.5.RELEASE&lt;/org.spring.version&gt;
      &lt;webflow.version&gt;2.3.1.RELEASE&lt;/webflow.version&gt;
  &lt;/properties&gt;


  &lt;dependencies&gt;
    &lt;!-- Spring --&gt;
    &lt;dependency&gt;
      &lt;groupId&gt;org.springframework&lt;/groupId&gt;
      &lt;artifactId&gt;spring-core&lt;/artifactId&gt;
      &lt;version&gt;${org.spring.version}&lt;/version&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
      &lt;groupId&gt;org.springframework&lt;/groupId&gt;
      &lt;artifactId&gt;spring-expression&lt;/artifactId&gt;
      &lt;version&gt;${org.spring.version}&lt;/version&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
      &lt;groupId&gt;org.springframework&lt;/groupId&gt;
      &lt;artifactId&gt;spring-beans&lt;/artifactId&gt;
      &lt;version&gt;${org.spring.version}&lt;/version&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
      &lt;groupId&gt;org.springframework&lt;/groupId&gt;
      &lt;artifactId&gt;spring-aop&lt;/artifactId&gt;
      &lt;version&gt;${org.spring.version}&lt;/version&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
      &lt;groupId&gt;org.springframework&lt;/groupId&gt;
      &lt;artifactId&gt;spring-context&lt;/artifactId&gt;
      &lt;version&gt;${org.spring.version}&lt;/version&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
      &lt;groupId&gt;org.springframework&lt;/groupId&gt;
      &lt;artifactId&gt;spring-web&lt;/artifactId&gt;
      &lt;version&gt;${org.spring.version}&lt;/version&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
      &lt;groupId&gt;org.springframework&lt;/groupId&gt;
      &lt;artifactId&gt;spring-webmvc&lt;/artifactId&gt;
      &lt;version&gt;${org.spring.version}&lt;/version&gt;
    &lt;/dependency&gt;
    
    &lt;!-- Spring Webflow --&gt;
    &lt;dependency&gt;
      &lt;groupId&gt;org.springframework.webflow&lt;/groupId&gt;
      &lt;artifactId&gt;spring-webflow&lt;/artifactId&gt;
      &lt;version&gt;${webflow.version}&lt;/version&gt;
    &lt;/dependency&gt;
  &lt;/dependencies&gt;</pre>
  
  <p>
    &nbsp;
  </p>
  
  <h2>
    <span style="color: #073763;">Configuring Spring</span>
  </h2>
</div>

<span style="font-family: Calibri;">I&#8217;ve divided the configuration, which consists in Spring configuration and integration with webflow, into three files, which are the following:</span>

  *  **root-context.xml**: The parent context, loaded by the Spring listener. This file contains back end configuration like repositories or business services. This tutorial is quite simple and I just pretend to explain how to configure Webflow, so this file will be empty.
  * **app-context.xml**: The child context which extends the root context. This context contains the web layer configuration like view resolvers, beans and controllers.
  * **webflow-config.xml**: This file configures webflow and it will be imported by the application context (app-context.xml). It could go inside the application context file but I keep it separate because I prefer to have the configuration structured.

The content of the above files is described  below:

<div style="margin: 0cm 0cm 0pt;">
  <span style="font-family: Calibri;"><b>app-context.xml</b></span>
</div>

<div style="line-height: normal; margin: 0cm 0cm 0pt; mso-layout-grid-align: none;">
  <pre class="lang:xhtml decode:true ">&lt;?xml version="1.0"encoding="UTF-8"?&gt;
&lt;beans xmlns=http://www.springframework.org/schema/beans
      xmlns:xsi=http://www.w3.org/2001/XMLSchema-instance
      xmlns:mvc=http://www.springframework.org/schema/mvc
      xmlns:context=http://www.springframework.org/schema/context
      xsi:schemaLocation="
            http://www.springframework.org/schema/beans 
            http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
            http://www.springframework.org/schema/context 
            http://www.springframework.org/schema/context/spring-context-3.0.xsd
            http://www.springframework.org/schema/mvc 
            http://www.springframework.org/schema/mvc/spring-mvc-3.0.xsd"&gt;

      &lt;import resource="webflow-config.xml"/&gt;
    
      &lt;!-- Detects mvc annotations like @RequestMapping --&gt;
      &lt;mvc:annotation-driven/&gt;

      &lt;!-- Detects @Component, @Service, @Controller, @Repository, @Configuration --&gt;
      &lt;context:component-scan base-package="org.spring.tutorial.controller"/&gt;


      &lt;bean class="org.springframework.web.servlet.view.InternalResourceViewResolver"&gt;
            &lt;property name="prefix"value="/WEB-INF/views/"/&gt;
            &lt;property name="suffix"value=".jsp"/&gt;
      &lt;/bean&gt;
&lt;/beans&gt;</pre>
</div>

<div style="margin: 0cm 0cm 0pt;">
  <p>
    <span style="font-family: Calibri;"> I haven&#8217;t included any controller definition because I prefer to use annotations. The component-scan tag will be responsible of scanning the base package to look for classes annotated with @Component and @Controller among others. In this tutorial I&#8217;ve annotated my controllers with @Controller annotation. You should keep in mind that component-scan tag will also search the classpath, so it is not recommended to set the base-package with values like &#8220;com&#8221; or &#8220;org&#8221;.</span>
  </p>
  
  <p>
    <span style="font-family: Calibri;">The view resolver is used by Spring Web MVC requests, and is responsible of resolving view logic names returned by the MVC controllers.</span>
  </p>
</div>

## <span style="color: #003366;">Configuring Webflow</span>

<div style="margin: 0cm 0cm 0pt;">
  <span style="font-family: Calibri;"><b>webflow-config.xml</b></span>
</div>

<div style="line-height: normal; margin: 0cm 0cm 0pt; mso-layout-grid-align: none;">
  <pre class="lang:xhtml decode:true ">&lt;?xml version="1.0"encoding="UTF-8"?&gt;
&lt;beans xmlns=http://www.springframework.org/schema/beans
      xmlns:xsi=http://www.w3.org/2001/XMLSchema-instance
      xmlns:webflow=http://www.springframework.org/schema/webflow-config
      xsi:schemaLocation="
            http://www.springframework.org/schema/beans 
            http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
            http://www.springframework.org/schema/webflow-config
        http://www.springframework.org/schema/webflow-config/spring-webflow-config-2.0.xsd"&gt;

      &lt;bean class="org.springframework.webflow.scope.ScopeRegistrar"/&gt;
      
      &lt;!-- Spring Webflow central configuration component --&gt;
      &lt;webflow:flow-executor id="flowExecutor"flow-registry="flowRegistry"/&gt;
    
      &lt;webflow:flow-registry id="flowRegistry"&gt;
          &lt;webflow:flow-location path="/WEB-INF/flows/main/main.xml"/&gt;
          &lt;!-- Could use a pattern instead 
          &lt;webflow:flow-location-pattern value="/**/*-flow.xml"/&gt;
          --&gt;
      &lt;/webflow:flow-registry&gt;

      &lt;bean class="org.springframework.webflow.mvc.servlet.FlowHandlerAdapter"&gt;
          &lt;property name="flowExecutor"ref="flowExecutor" /&gt;
      &lt;/bean&gt;

      &lt;bean class="org.springframework.webflow.mvc.servlet.FlowHandlerMapping"&gt;
          &lt;property name="flowRegistry"ref="flowRegistry"/&gt;
          &lt;property name="order"value="0"/&gt;
      &lt;/bean&gt;
&lt;/beans&gt;</pre>
  
  <p>
    &nbsp;
  </p>
</div>

**ScopeRegistar**: Registers webflow scopes. This allows you to use them in addition to the usual Spring scopes. These scopes are flash, view, flow and conversation.

<span style="font-family: Calibri;"><b>Flow registry</b>: Registers all the flows that will be used in the web application. It is required to specify where is every flow definition located, which means the xml file. In this tutorial, we specify the exact location of the file with the flow-location property, but you could instead specify a pattern.</span>

<span style="font-family: Calibri;"><b>FlowHandlerAdapter</b>: Activates the flow management with Spring MVC.</span>

<div style="margin: 0cm 0cm 0pt;">
  <span style="font-family: Calibri;"><b>FlowHandlerMapping</b>: The first mapping that will be invoked. It will check if the request path maps with the id of a flow registered in the flowRegistry. If it finds it, the flow will be started. If not, will invoke the next mapping. On this example, you will define a request mapping for an MVC handler.</span>
</div>

&nbsp;

## <span style="color: #073763;">Configuring the webapp</span>

The web.xml is as follows:

<pre class="lang:default decode:true ">&lt;!-- root context configuration --&gt;
  &lt;context-param&gt;
      &lt;param-name&gt;contextConfigLocation&lt;/param-name&gt;
      &lt;param-value&gt;/WEB-INF/config/root-context.xml&lt;/param-value&gt;
  &lt;/context-param&gt;

  &lt;!-- Loads the root context into the servletContext before any servlets are initialized --&gt;
  &lt;listener&gt;
      &lt;listener-class&gt;org.springframework.web.context.ContextLoaderListener&lt;/listener-class&gt;
  &lt;/listener&gt;

  &lt;!-- Spring servlet --&gt;
  &lt;servlet&gt;
      &lt;servlet-name&gt;springServlet&lt;/servlet-name&gt;
      &lt;servlet-class&gt;org.springframework.web.servlet.DispatcherServlet&lt;/servlet-class&gt;
      &lt;init-param&gt;
            &lt;param-name&gt;contextConfigLocation&lt;/param-name&gt;
            &lt;param-value&gt;/WEB-INF/config/app-context.xml&lt;/param-value&gt;
      &lt;/init-param&gt;
  &lt;/servlet&gt;
  &lt;servlet-mapping&gt;
      &lt;servlet-name&gt;springServlet&lt;/servlet-name&gt;
      &lt;url-pattern&gt;/spring/*&lt;/url-pattern&gt;
  &lt;/servlet-mapping&gt;</pre>

<div style="margin: 0cm 0cm 0pt;">
</div>

<h2 style="margin: 0cm 0cm 0pt;">
  <span style="color: #073763;">Implementing MVC handler</span>
</h2>

<span style="font-family: Calibri;">The MVC mapping is configured with annotations (@RequestMapping). </span>

<div style="margin: 0cm 0cm 0pt;">
  <pre class="lang:java decode:true">import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
public class MvcController {

      @RequestMapping("/viewMvc")
      public String getData(Model model) {
            model.addAttribute("message", "Hello world MVC");
            return "mvcView";
      }
}</pre>
</div>

## <span style="color: #073763;">Building the flow</span>

The flow is really simple. It consists of an initial view which takes an input field, a navigation to an action state which will build the message, and a final view where the message will be shown.

This flow must be defined in the flow registry in order to be found. We did this in the webflow-config.xml. The following is the detail of the main.xml file:

<pre class="lang:xhtml decode:true ">&lt;view-state id="startPage"&gt;
            &lt;transition on="next"to="setMessage"/&gt;
      &lt;/view-state&gt;

      &lt;action-state id="setMessage"&gt;
            &lt;evaluate expression="setMessageController"/&gt;
            &lt;transition on="ok"to="finalView"/&gt;
      &lt;/action-state&gt;

      &lt;end-state id="finalView" view="finalView.jsp"/&gt;</pre>

The initial view (startPage) is composed of a form and a submit button which will take us to the next state. It is not necessary to specify the &#8216;view&#8217; attribute if the physical view name is the same as the id of the view-state.<span style="color: black; font-family: 'Courier New'; font-size: 10pt;"><span style="mso-tab-count: 1;">     </span></span>

<pre class="lang:default decode:true ">&lt;form method="post"&gt;

            Enter your name: &lt;inputtype="text" name="inputName"value=""/&gt;
            &lt;input type="hidden"name="_flowExecutionKey" value="${flowExecutionKey}"/&gt;
            &lt;input type="submit"class="button" name="_eventId_next"value="Next view"/&gt;
      &lt;/form&gt;</pre>

&nbsp;

You need to send a request parameter named &#8216;flowExecutionKey&#8217; when you submit the form. This way, when a request comes in with this parameter, Webflow will be able to resume the current flow execution and navigate to the next state.

Specifying &#8220;\_eventId\_next&#8221; as the name attribute on the component that executes the action, will launch the event &#8220;next&#8221;, which will execute the transition to the next state defined in the flow definition. You could also do this with a hidden field:<span style="color: black; font-family: 'Courier New'; font-size: 10pt;"><span style="color: black; font-family: 'Courier New'; font-size: 10pt;"><span style="mso-tab-count: 1;">     </span></span></span>

<pre class="lang:xhtml decode:true">&lt;input type="hidden"name="_eventId" value="next"/&gt;</pre>

&nbsp;

<div style="margin: 0cm 0cm 0pt;">
  <span style="font-family: Calibri;"> The next state &#8220;setMessage&#8221; will invoke the controller that will build the message. This controller must implement Action interface:</span>
</div>

<div style="line-height: normal; margin: 0cm 0cm 0pt; mso-layout-grid-align: none;">
  <pre class="lang:java decode:true ">import org.springframework.stereotype.Controller;
import org.springframework.webflow.execution.Action;
import org.springframework.webflow.execution.Event;
import org.springframework.webflow.execution.RequestContext;

@Controller("setMessageController")
public classWebflowController implements Action {

      @Override
      public Event execute(RequestContext req) throws Exception {
            String name = req.getRequestParameters().get("inputName");
            req.getFlowScope().put("message", "Hello "+name);
            return new Event(this, "ok");
      }
}</pre>
  
  <p>
    &nbsp;
  </p>
</div>

<div style="margin: 0cm 0cm 0pt;">
  <span style="font-family: Calibri;">Finally, we will show the message at the final view. This is done with Expression Language: </span><span style="font-family: Calibri;"> ${message}</span>
</div>

<div style="margin: 0cm 0cm 0pt;">
</div>

<div>
</div>

<h2 style="margin: 0cm 0cm 0pt;">
  <span style="color: #073763;">Accessing the webapp</span>
</h2>

<div style="margin: 0cm 0cm 0pt;">
  <span style="font-family: Calibri;">This url will be handled by Spring Web MVC and invoke the MVC controller:</span>
</div>

<span style="color: blue; font-family: Calibri;">http://localhost:8080/myWebapp/spring/viewMvc</span>

This url will be handled by Spring Webflow and will start the main flow:

<div style="margin: 0cm 0cm 0pt;">
  <span style="color: blue; font-family: Calibri;">http://localhost:8080/myWebApp/spring/main</span><u></u>
</div>

&nbsp;

## <span style="color: #073763;">Project structure</span>

<div style="clear: both; text-align: left;">
</div>

<div style="clear: both; text-align: center;">
  <a href="http://4.bp.blogspot.com/-DMVVuPHI66Q/VUY4tkUI1ZI/AAAAAAAAEic/l75eU5bn8GI/s1600/estructuraProjecte.png"><img loading="lazy" class="alignnone" src="http://4.bp.blogspot.com/-DMVVuPHI66Q/VUY4tkUI1ZI/AAAAAAAAEic/l75eU5bn8GI/s1600/estructuraProjecte.png" alt="spring-webflow-project-structure" width="211" height="273" border="0" /></a>
</div>

<div style="clear: both; text-align: left;">
</div>

As specified in the flow registry, flows are located in the WEB-INF/flows/ folder. The main folder contains the flow definition and the views associated to the flow.

The views folder contains MVC views. The location is specified at the view resolver.

## <span style="color: #003366;">Related posts</span>

  * [Testing Webflow 2 with inheritance](http://xpadro.com/2013/01/12/testing-webflow-2-with-inheritance/)

&nbsp;