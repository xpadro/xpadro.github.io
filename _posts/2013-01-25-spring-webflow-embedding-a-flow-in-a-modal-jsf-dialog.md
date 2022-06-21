---
id: 44
title: 'Spring Webflow: Embedding a flow in a modal JSF dialog'
date: 2013-01-25T21:18:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/01/25/spring-webflow-embedding-a-flow-in-a-modal-jsf-dialog/
permalink: /2013/01/spring-webflow-embedding-a-flow-in-a-modal-jsf-dialog.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/01/spring-webflow-embedding-flow-in-modal.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/6680929340564778935
image: /wp-content/uploads/2013/01/spring-webflow-logo.png
categories:
  - Spring
  - Spring Webflow
tags:
  - JSF
  - Spring
  - Webflow
---

The latest version of Spring Webflow (2.3.1) provides us with a new and very interesting functionality that I have really missed in my current project: embedded flows. By default, Webflow applies the <a href="http://en.wikipedia.org/wiki/Post/Redirect/Get" target="_blank" rel="noopener">POST/REDIRECT/GET</a> pattern every time it enters a view state. The reason to do this is to prevent duplicate form submissions, but this also prevents us from having two different view states on the same page.

With embedded flows, we have the possibility to execute transitions and load view states via Ajax requests, avoiding a full page render. Thanks to the <a href="https://src.springframework.org/svn/spring-samples/" target="_blank" rel="noopener">Spring samples</a> repository, I&#8217;ve had the opportunity to see how to embed a flow on a page. In this tutorial, I will explain how to apply the example within a modal dialog.

You can get this tutorial source code at github: <a href="https://github.com/xpadro/spring-webflow" target="_blank" rel="noopener">https://github.com/xpadro/spring-webflow</a>

## <span style="color: #073763;">The flow</span>

The demo application consists of a main flow called &#8216;sales-flow&#8217;, which shows the sales made by a specified employee. The employee information view contains an access to a subflow called &#8217;employee-flow&#8217; that loads the employee data. This subflow will fully execute within a modal dialog.

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://1.bp.blogspot.com/-RckfCtvNQRE/VUY6LU7D2iI/AAAAAAAAEiw/5-WvFe4ZvpY/s1600/pic1.png"><img loading="lazy" class="alignnone" src="http://1.bp.blogspot.com/-RckfCtvNQRE/VUY6LU7D2iI/AAAAAAAAEiw/5-WvFe4ZvpY/s1600/pic1.png" alt="spring webflow flow diagram" width="320" height="299" border="0" /></a>
</div>

&nbsp;

### <span style="color: #073763;"><span style="background-color: white;">How it works</span></span>

In order to execute a flow in embedded mode, you simply need to add a parameter to the request url. For example:

<div style="margin-bottom: 0cm;">
  <span style="mso-tab-count: 1;">                </span><a href="http://localhost:8080/myApp/myFlow?mode=embedded">http://localhost:8080/myApp/myFlow?mode=embedded</a>
</div>

As we need to call the subflow from the main sales flow, we can also do it by defining an input attribute.

In this example, the main flow calls the subflow using an input attribute:

<div style="line-height: normal; margin-bottom: 0cm; mso-layout-grid-align: none; text-autospace: none;">
  <p>
    <span style="color: black; font-family: 'Courier New'; font-size: 10.0pt;"><span style="color: black; font-family: 'Courier New'; font-size: 10.0pt;"><span style="mso-tab-count: 1;">     </span></span></span>
  </p>
  
  <pre class="lang:xhtml decode:true ">&lt;view-state id="main"&gt;
            &lt;transition on="start"to="embedded-flow"/&gt;
            &lt;transition on="next"to="sales"/&gt;
      &lt;/view-state&gt;
     
      &lt;view-state id="sales"&gt;
            &lt;transition on="back"to="main"/&gt;
      &lt;/view-state&gt;

      &lt;subflow-state id="embedded-flow"subflow="sales-flow/employee-flow"&gt;
            &lt;input name="mode"value="'embedded'"/&gt;
            &lt;transition on="final"to="main"/&gt;
      &lt;/subflow-state&gt;</pre>
  
  <p>
    And the embedded flow:<span style="color: black; font-family: 'Courier New'; font-size: 10pt;">     </span>
  </p>
</div>

<div style="line-height: normal; margin-bottom: 0cm; mso-layout-grid-align: none; text-autospace: none;">
  <pre class="lang:xhtml decode:true">&lt;action-state id="getDataAction"&gt;
            &lt;evaluate expression="getDataController"/&gt;
            &lt;transition on="yes"to="confirmation"/&gt;
      &lt;/action-state&gt;
     
      &lt;view-state id="confirmation"&gt;
            &lt;transition on="reset"to="reset"/&gt;
            &lt;transition on="confirm"to="final"/&gt;
      &lt;/view-state&gt;

      &lt;view-state id="reset"&gt;
            &lt;transition on="getData"to="getDataAction"/&gt;
      &lt;/view-state&gt;
     
      &lt;end-state id="final"/&gt;</pre>
  
  <p>
    &nbsp;
  </p>
</div>

To load the embedded flow on the main page of the sales flow, I use the dialog component from the Primefaces library:

<div style="line-height: normal; margin-bottom: 0cm; mso-layout-grid-align: none; text-autospace: none;">
  <pre class="lang:xhtml decode:true ">&lt;p:dialog widgetVar="employeeSearch"header="Search employee" modal="true" width="580"&gt;
   &lt;p:outputPanel&gt;
      &lt;h:form id="mainForm"&gt;
         Insert employee name and surname: &lt;br /&gt;&lt;br /&gt;
         Name: &lt;h:inputText value="#{userBean.name}"/&gt;
         Surname: &lt;h:inputText value="#{userBean.surname}"/&gt;&lt;br /&gt;&lt;br /&gt;
         &lt;p style="text-align:center"&gt;
            &lt;p:commandButton value="Get data"action="start" update="mainForm"/&gt;
         &lt;/p&gt;
      &lt;/h:form&gt;  
   &lt;/p:outputPanel&gt;
&lt;/p:dialog&gt;</pre>
  
  <p>
    &nbsp;
  </p>
</div>

The commandButton component will execute the action &#8216;start&#8217; that will start the embedded flow as a subflow. The main view of the embedded flow is as follows:

<div style="line-height: normal; margin-bottom: 0cm; mso-layout-grid-align: none; text-autospace: none;">
  <pre class="lang:xhtml decode:true ">&lt;ui:composition xmlns="http://www.w3.org/1999/xhtml"
                xmlns:ui="http://java.sun.com/jsf/facelets"
                xmlns:h="http://java.sun.com/jsf/html"
                xmlns:f="http://java.sun.com/jsf/core"
                xmlns:p="http://primefaces.org/ui"
                template="/WEB-INF/layouts/standard.xhtml"&gt;

&lt;ui:define name="content"&gt;
   &lt;h:form id="mainForm"&gt;
      Employee data:&lt;br /&gt;
      &lt;p:fieldset&gt;
        &lt;h:outputLabel style="font-family:bold"for="name"&gt;Name: &lt;/h:outputLabel&gt;
        &lt;h:outputText id="name"value="#{userBean.name}"/&gt;&lt;br/&gt;
        &lt;h:outputLabel style="font-family:bold"for="surname"&gt;Surname: &lt;/h:outputLabel&gt;
        &lt;h:outputText id="surname"value="#{userBean.surname}"/&gt;
      &lt;/p:fieldset&gt;
           
      &lt;br/&gt;
      Additional info:&lt;br /&gt;
      &lt;p:fieldset&gt;
        &lt;h:outputLabel style="font-family:bold"for="code"&gt;Code: &lt;/h:outputLabel&gt;
        &lt;h:outputText id="code"value="#{userBean.code}"/&gt;&lt;br/&gt;
        &lt;h:outputLabel style="font-family:bold"for="city"&gt;City: &lt;/h:outputLabel&gt;
        &lt;h:outputText id="city"value="#{userBean.city}"/&gt;
      &lt;/p:fieldset&gt;

      &lt;p&gt;
        Confirm selection?
      &lt;/p&gt;
      &lt;p style="text-align:center"&gt;
        &lt;p:commandButton value="Reset"action="reset" update="mainForm"/&gt;
        &lt;p:commandButton value="Confirm"action="confirm" update="mainForm"/&gt;  
      &lt;/p&gt;
   &lt;/h:form&gt;
&lt;/ui:define&gt;
&lt;/ui:composition&gt;</pre>
  
  <p>
    &nbsp;
  </p>
</div>

Keep in mind that the form must have the same id attribute in all the embedded flow views. Otherwise, it won&#8217;t be possible to execute the embedded flow.

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://3.bp.blogspot.com/-QliU5p9vIk8/VUY6NOwbvjI/AAAAAAAAEi4/DUKZYZlKw2o/s1600/pic2.png"><img loading="lazy" class="alignnone" src="http://3.bp.blogspot.com/-QliU5p9vIk8/VUY6NOwbvjI/AAAAAAAAEi4/DUKZYZlKw2o/s1600/pic2.png" alt="spring webflow views" width="640" height="241" border="0" /></a>
</div>

&nbsp;

## Related posts

  * [Configure Spring Webflow 2 with JSP views](http://xpadro.com/2012/12/04/configure-webflow-2-with-jsp-views/)
  * [Testing Webflow 2 with inheritance](http://xpadro.com/2013/01/12/testing-webflow-2-with-inheritance/)

&nbsp;