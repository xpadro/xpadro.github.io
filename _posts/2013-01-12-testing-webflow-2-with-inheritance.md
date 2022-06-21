---
id: 45
title: Testing Spring Webflow 2 with inheritance
date: 2013-01-12T17:20:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/01/12/testing-webflow-2-with-inheritance/
permalink: /2013/01/testing-webflow-2-with-inheritance.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/01/testing-webflow-2-with-inheritance.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/7469936956996304088
image: /wp-content/uploads/2013/01/spring-webflow-logo.png
categories:
  - Spring
  - Spring Webflow
tags:
  - Spring
  - Webflow
---
<span style="font-family: Calibri;">This blog entry shows how to test a flow with inheritance in Spring Webflow 2. The flow to be tested consists of a simple navigation which starts with a view state and ends getting to another view state that will depend on the result of the execution of a controller. This flow extends another flow which basically contains a redirection to a common page in case of error.</span>

<h2 style="margin: 0cm 0cm 0pt;">
  <span style="color: #073763;">Introduction</span>
</h2>

<div style="margin: 0cm 0cm 0pt;">
  <span style="font-family: Calibri;">The test flow (main.xml) is as follows:</span>
</div>

<div style="line-height: normal; margin: 0cm 0cm 0pt; mso-layout-grid-align: none;">
  <pre class="lang:xhtml decode:true">&lt;?xml version="1.0"encoding="UTF-8"?&gt;
&lt;flow xmlns="http://www.springframework.org/schema/webflow"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.springframework.org/schema/webflow http://www.springframework.org/schema/webflow/spring-webflow-2.0.xsd"
      parent="parent"&gt;

      &lt;view-state id="startPage"view="main/startPage.jsp"&gt;
            &lt;transition on="next"to="generateError"/&gt;
      &lt;/view-state&gt;
      
      &lt;action-state id="generateError"&gt;
            &lt;evaluate expression="checkParameterController"/&gt;
            &lt;transition on="ok"to="OkView"/&gt;
            &lt;transition on="ko"to="KoView"/&gt;
      &lt;/action-state&gt;


      &lt;end-state id="OkView"view="main/finalOkView.jsp"/&gt;
      &lt;end-state id="KoView"view="main/finalKoView.jsp"/&gt;
&lt;/flow&gt;</pre>
  
  <p>
    &nbsp;
  </p>
  
  <p>
    <span style="font-family: Calibri;">And the parent flow (parent.xml):</span>
  </p>
</div>

<div style="margin: 0cm 0cm 0pt;">
  <pre class="lang:xhtml decode:true">&lt;?xml version="1.0"encoding="UTF-8"?&gt;
&lt;flow xmlns="http://www.springframework.org/schema/webflow"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.springframework.org/schema/webflow http://www.springframework.org/schema/webflow/spring-webflow-2.0.xsd"
      abstract="true"&gt;


      &lt;end-state id="errorState"view="commonErrorPage.jsp"/&gt;


      &lt;global-transitions&gt;
            &lt;transition on-exception="java.lang.Exception"to="errorState"/&gt;
      &lt;/global-transitions&gt;
&lt;/flow&gt;</pre>
  
  <p>
    &nbsp;
  </p>
</div>

<div style="margin: 0cm 0cm 0pt;">
  When executing tests, Spring provides us with a quite useful class: <i>AbstractXmlFlowExecutionTests</i>, in the <i>org.springframework.webflow.test.execution</i> package. This class has a variety of methods that will help us test the flow. The most interesting:
</div>

<div style="margin: 0cm 0cm 0pt;">
</div>

<div style="margin: 0cm 0cm 0pt;">
  <pre class="lang:java decode:true">FlowDefinitionResource getResource(FlowDefinitionResourceFactory resourceFactory)</pre>
  
  <p>
    &nbsp;
  </p>
</div>

<span style="font-family: Calibri;">Here, we specify where the flow which we want to test is located.</span>

<div style="line-height: normal; margin: 0cm 0cm 0pt; mso-layout-grid-align: none;">
  <pre class="lang:java decode:true">FlowDefinitionResource[] getModelResources(FlowDefinitionResourceFactory resourceFactory)</pre>
  
  <p>
    &nbsp;
  </p>
</div>

<span style="font-family: Calibri;">If the flow uses inheritance, we define the parent flow here.</span>

<pre class="lang:java decode:true">void configureFlowBuilderContext(MockFlowBuilderContext builderContext)</pre>

Lets us customize the builder context. I use it for registering the beans that will use in the test as this is the way it&#8217;s indicated at the class javadoc.

<pre class="lang:java decode:true">void registerMockFlowBeans(ConfigurableBeanFactory flowBeanFactory)</pre>

Allows registering the beans that will be used by the flow to be tested. For example:

<pre class="lang:java decode:true ">flowBeanFactory.registerSingleton("myService", new MyMockService());</pre>

&nbsp;

<h2 style="line-height: normal; margin: 0cm 0cm 0pt; mso-layout-grid-align: none;">
  <span style="color: #073763;">Testing the flow</span>
</h2>

I&#8217;ve divided it in two classes in order to separate configuration from tests cases:

<pre class="lang:java decode:true ">public classBaseTestFlow  extendsAbstractXmlFlowExecutionTests {
      protected ApplicationContext applicationContext;


      /**
       * This method returns the flow to be tested.
       */
      protectedFlowDefinitionResource getResource(FlowDefinitionResourceFactory resourceFactory) {
            returnresourceFactory.createFileResource("src/test/resources/flows/main.xml");
      }


      /**
       * Needs to be overridden if the flow to be tested inherits from another flow. In this case, we register its parent flow.
       */
      protectedFlowDefinitionResource[]
       getModelResources(FlowDefinitionResourceFactory resourceFactory) {
        FlowDefinitionResource[] flowDefinitionResources = new FlowDefinitionResource[1];
        flowDefinitionResources[0] = resourceFactory.createFileResource("src/test/resources/flows/parent.xml");
        returnflowDefinitionResources;
      }


      /**
       * Registers beans used by the tested flow.
       */
      protected voidconfigureFlowBuilderContext(MockFlowBuilderContext builderContext) {
            applicationContext = newClassPathXmlApplicationContext(new String[] {
                  "classpath:xpadro/spring/test/configuration/root-context.xml",
                  "classpath:xpadro/spring/test/configuration/app-context.xml"
            });
            builderContext.registerBean("checkParameterController", applicationContext.getBean("checkParameterController"));
      }


      /*
      protected void registerMockFlowBeans(ConfigurableBeanFactory flowBeanFactory) {
            flowBeanFactory.registerSingleton("checkParameterController", new CheckParameterController());
      }
      */
}</pre>

&nbsp;

You could simply delete configureFlowBuilderContext method and use registerMockFlowBeans method instead if you don&#8217;t want/need to start your own test context.

<div style="line-height: normal; margin: 0cm 0cm 0pt; mso-layout-grid-align: none;">
  <pre class="lang:java decode:true ">public class TestFlow extends BaseTestFlow {
      public voidtestFlowStarts() {
            MockExternalContext externalContext = new MockExternalContext();
            startFlow(externalContext);
            assertFlowExecutionActive();
            assertCurrentStateEquals("startPage");
      }


      public voidtestNavigation() {
            MockExternalContext externalContext = new MockExternalContext();
            setCurrentState("startPage");
            externalContext.setEventId("next");
            externalContext.getMockRequestParameterMap().put("inputField", "yes");
            getFlowScope().put("testFlowVar", "testValue");
            resumeFlow(externalContext);
            assertCurrentStateEquals("OkView");
            assertEquals("testValue", getFlowScope().get("testFlowVar"));
      }


      public voidtestGlobalTransition() {
            MockExternalContext externalContext = new MockExternalContext();
            setCurrentState("startPage");
            externalContext.setEventId("next");
            externalContext.getMockRequestParameterMap().put("inputField", "error");
            resumeFlow(externalContext);
            assertFlowExecutionOutcomeEquals("errorState");
      }
}</pre>
</div>

## <span style="color: #003366;">Related posts</span>

  * [Configure Spring Webflow 2 with JSP views](http://xpadro.com/2012/12/04/configure-webflow-2-with-jsp-views/)