---
id: 39
title: Communication in Spring Webflow 2
date: 2013-04-03T21:40:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/04/03/communication-in-spring-webflow-2/
permalink: /2013/04/communication-in-spring-webflow-2.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/04/communication-in-spring-webflow-2.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/7807041444670459139
categories:
  - Spring
  - Spring Webflow
tags:
  - Spring
  - Webflow
---

This article tries to complement the reference documentation with examples on variables, scopes and flows in Spring Webflow 2. It shows different ways to share data between controllers and views that form a flow. The article is divided into the following sections:

  * Setting flow variables
  * Setting attribute values
  * Using additional scopes
  * Communication with sub flows
  * Communication with other flows
  * Launching events with attributes
  * Accessing RequestContext
  * Accessing Spring beans
  * Using implicit variables

The example explained here uses Spring Web flow 2.3.1 and Spring 3.0.5 versions. You can get the source code <a href="https://github.com/xpadro/spring-webflow/tree/master/sp_webflow_communication" target="_blank" rel="noopener">my github webflow repository</a>, with all the examples shown in this article.

## <span style="color: #073763;">1   Setting flow variables</span>

There are different ways of setting variables and storing them into the flow scope, where they will then be accessible to controllers and views.

### <span style="color: #073763;">1.1 var element (variable)</span>

The _var_ element instantiates a class and stores it in the flow scope. Since the instance will be stored between flow requests, the class should implement java.io.Serializable. It does not allow you to assign a value, making this element useless for classes without a default constructor (it will crash at runtime with a nice NoSuchMethodException). In that case, you can use the _set_ element (see section 3).

Defining a variable:

<pre class="lang:xhtml decode:true ">&lt;var name="car" class="xpadro.tutorial.webflow.model.Car"/&gt;</pre>

You can use this variable in the flow definition, for example, passing it as a parameter to a controller:

<pre class="lang:xhtml decode:true ">&lt;var name="car" class="xpadro.tutorial.webflow.model.Car"/&gt;

&lt;action-state id="startAssembly"&gt;
    &lt;evaluate expression="carFactory.paintCar(car)"/&gt;
	...</pre>

The method will receive the car parameter:

<pre class="lang:xhtml decode:true ">public Event paintCar(Car car) {
    ...</pre>

Since it is automatically stored in the flow scope, you can also retrieve the value from the request context:

<pre class="lang:xhtml decode:true ">public Event myControllerMethod(RequestContext context) {
    Car car = (Car) context.getFlowScope().get("car");
    ...</pre>

Or use it in the view using for example ${car.color}

### <span style="color: #073763;">1.2 At flow starting point</span>

If your class has no default constructor or you want to retrieve the instance from a service, the element _var_ won&#8217;t be enough. You can use a controller instead:

<pre class="lang:xhtml decode:true ">&lt;on-start&gt;
    &lt;evaluate expression="customerService.getPreferences()" result="flowScope.preferences"/&gt;
&lt;/on-start&gt;</pre>

In this sample, the information could be retrieved from a service. The result attribute will get the object returned by the controller and store it in the flow scope.

You can also use this approach when entering a state (on-entry) or before rendering a view (on-render).

### <span style="color: #073763;">1.3 Anywhere in the flow</span>

Since you have access to the Web flow RequestContext when invoking a controller, you can set a flow scoped variable as shown below:

<pre class="lang:xhtml decode:true ">public Event myControllerMethod(RequestContext context) {
    context.getFlowScope().put("car", new Car());
    ...</pre>

&nbsp;

## <span style="color: #073763;">2 Setting attribute values</span>

The _set_ element allows you to define an attribute and specify a scope. This element takes the following attributes:

  * name: Scope and name for the attribute, delimited by a dot (scope.name).
  * value: Value set to the attribute.
  * type: Class type.

For example, you can use it at the beginning of a flow (on-start) or when entering into a state (on-entry). The following sample show how to assign it a specified value when the flow starts:

<pre class="lang:xhtml decode:true ">&lt;on-start&gt;
    &lt;set name="flowScope.factoryId" value="5061" type="java.lang.Integer"/&gt;
&lt;/on-start&gt;</pre>

Or you could set it when launching a transition:

<pre class="lang:xhtml decode:true ">&lt;transition on="success" to="nextState"&gt;
    &lt;set name="flowScope.factoryId" value="5061" type="java.lang.Integer"/&gt;
&lt;/transition&gt;</pre>

The _set_ element not only allows you to define objects, but also String values. For example, if you have a bean named &#8216;myBean&#8217;, the first _set_ element in the following sample will retrieve the Car instance from the bean. Then, it will store it in the request scope with the name &#8216;carObject&#8217;. On the other hand, the second _set_ element will store in the request scope an attribute named &#8216;carString&#8217;. This attribute contains the String &#8216;myBean.car&#8217;.

<pre class="lang:xhtml decode:true ">&lt;transition on="success" to="addMechanics"&gt;
    &lt;set name="requestScope.carObject" value="myBean.car"/&gt;
    &lt;set name="requestScope.carString" value="'myBean.car'"/&gt;
&lt;/transition&gt;</pre>

You can also use implicit variables when setting the value of the _set_ element (see section 10 for a list of these variables):

<pre class="lang:xhtml decode:true ">&lt;set name="requestScope.carBean" value="flowScope.car"/&gt;</pre>

&nbsp;

## <span style="color: #073763;">3 Using additional scopes</span>

The RequestContext interface contains access to all the other scopes defined in Spring Web flow: request, flash, view, flow and conversation.

You can also access these scopes at flow definition level by using implicit EL variables, which are: requestScope, flashScope, viewScope, flowScope and conversationScope.

For other scopes, you can use the external context:

At flow definition using implicit variables:

<pre class="lang:xhtml decode:true ">&lt;set name="externalContext.sessionMap.factoryId" value="5061" type="java.lang.Integer"/&gt;</pre>

Or at controller level through RequestContext interface:

<pre class="lang:java decode:true ">context.getExternalContext().getSessionMap().put("factoryId", 5061);</pre>

&nbsp;

## <span style="color: #073763;">4 Communication with sub flows</span>

When invoking a sub flow from the main flow, you can pass it input attributes. Once the sub flow has finished, it may return output attributes.

**Input:**  
The _input_ element allows you to send parameters to the sub flow. When the sub flow starts, these input attributes are stored in the flow scope of the sub flow. You will need to define the _input_ element in the sub flow, using the same name attribute as used in the main flow.

**Output:**  
Once the sub flow ends, the main flow can receive output parameters. These output parameters are defined within sub flow&#8217;s end-states. When the execution returns to the main flow. output parameters will be available as attributes inside the launched event.

<u>Main flow: invoking a sub flow</u>

<pre class="lang:xhtml decode:true ">&lt;subflow-state id="addMechanics" subflow="mechanics-flow"&gt;
    &lt;input name="currentCar" value="requestScope.carInstance1" type="xpadro.tutorial.webflow.model.Car"/&gt;
    &lt;input name="preferences" value="flowScope.preferences" type="java.util.Map"/&gt;
    &lt;transition on="mechanicsSuccess" to="carValidationPhase1"&gt;
        &lt;set name="flowScope.mechanics" value="currentEvent.attributes.mechanics" /&gt;
    &lt;/transition&gt;
    &lt;transition on="mechanicsFail" to="assemblyFailed"/&gt;
&lt;/subflow-state&gt;</pre>

&nbsp;

When returning to the main flow, you will need to define an attribute with the value returned by the sub flow in order to make it accessible to following states.

You could also invoke a controller that would set the value and store it in the needed scope:

<pre class="lang:default decode:true ">&lt;transition on="mechanicsSuccess" to="assemblyFinalized"&gt;
    &lt;evaluate expression="carFactory.setOutcome(currentEvent.attributes.mechanics)" /&gt;  
&lt;/transition&gt;</pre>

<u>Sub flow definition</u>

<pre class="lang:xhtml decode:true ">&lt;input name="currentCar" type="xpadro.tutorial.webflow.model.Car"/&gt;
&lt;input name="preferences" type="java.util.Map"/&gt;
    
&lt;action-state id="setMechanics"&gt;
    &lt;evaluate expression="carFactory.addMechanics"/&gt;
    &lt;transition on="success" to="mechanicsSuccess"/&gt;
    &lt;transition on="fail" to="mechanicsFail"/&gt;
&lt;/action-state&gt;
	
&lt;end-state id="mechanicsSuccess"&gt;
    &lt;output name="mechanics" value="'success'"/&gt;
&lt;/end-state&gt;
	
&lt;end-state id="mechanicsFail"&gt;
    &lt;output name="mechanics" value="'fail'"/&gt;
&lt;/end-state&gt;</pre>

&nbsp;

There are other options to pass information to a subflow which consists in the following:

  * Storing the information at conversation scope (this scope is available from within a flow and its sub flows).
  * Pass the information in the transition which access the sub flow:

<pre class="lang:xhtml decode:true">&lt;transition on="success" to="addMechanics"&gt;
    &lt;set name="requestScope.carInstance" value="car"/&gt;
&lt;/transition&gt;</pre>

&nbsp;

## <span style="color: #073763;">5 Communication with other flows</span>

You have two options of passing data to another flow which is not related to the current flow:

  1. Session scoped attributes
  2. URL Request parameters

If you choose the second option, you can do it the following way:

In the view:

<pre class="lang:xhtml decode:true ">&lt;a href="otherflow-flow?otherParam=otherValue"&gt;other flow&lt;/a&gt;</pre>

When starting the other flow, you can use the requestParameters implicit variable to retrieve the value and store it in the needed scope.

<pre class="lang:xhtml decode:true ">&lt;on-start&gt;
    &lt;set name="flowScope.urlParam" value="requestParameters.otherParam" type="java.lang.String" /&gt;
&lt;/on-start&gt;</pre>

You could also do it in the controller:

<pre class="lang:xhtml decode:true ">&lt;on-start&gt;
    &lt;evaluate expression="startFlowController.start"/&gt;
&lt;/on-start&gt;</pre>

<pre class="lang:java decode:true ">public Event start(RequestContext context) {
    ParameterMap map = context.getExternalContext().getRequestParameterMap();
    context.getFlashScope().put("otherParam", map.get("otherParam"));
    ...</pre>

Or use the requestParameterMap directly through the getRequestParameters shortcut method:

<pre class="lang:java decode:true ">ParameterMap map = context.getRequestParameters();
context.getFlashScope().put("otherParam", map.get("otherParam"));</pre>

&nbsp;

## <span style="color: #073763;">6 Launching events with attributes</span>

When exiting a controller, it is possible to add attributes to the current event. To do that you need to generate an attribute map.

<pre class="lang:xhtml decode:true ">&lt;action-state id="action1"&gt;
    &lt;evaluate expression="myFirstController.setAttribute()"/&gt;
    &lt;transition on="done" to="action2"/&gt;
&lt;/action-state&gt;

&lt;action-state id="action2"&gt;
    &lt;evaluate expression="mySecondController.getAttribute(currentEvent.attributes.testAttribute)"/&gt;
    &lt;transition on="done" to="myView"/&gt;
&lt;/action-state&gt;</pre>

The controller which launches the event adds the attribute as follows:

<pre class="lang:java decode:true">public Event setAttribute() {
    Map&lt;String,String&gt; map = new HashMap&lt;String,String&gt;();
    map.put("testAttribute", "test");
    AttributeMap attributeMap = new LocalAttributeMap(map);
    return new Event(this, "success", attributeMap);
}

</pre>

&nbsp;

## <span style="color: #073763;">7 Accessing the request context</span>

If you want to invoke methods from classes other than controllers, like beans, it is possible to retrieve the web flow request context. There are two ways:

Passing the request context as a parameter to the bean method:

<pre class="lang:xhtml decode:true ">&lt;action-state id="carValidationPhase1"&gt;
    &lt;evaluate expression="supportBean.validateCarColor(flowRequestContext)"/&gt;
    &lt;transition on="success" to="carValidationPhase2"/&gt;
&lt;/action-state&gt;</pre>

<pre class="lang:java decode:true ">public String validateCarColor(RequestContext context) {
    Car car = (Car) context.getFlowScope().get("car");
    ...</pre>

Or you can use the RequestContextHolder class:

<pre class="lang:xhtml decode:true ">&lt;action-state id="carValidationPhase2"&gt;
    &lt;evaluate expression="supportBean.validateCarMechanics()"/&gt;
    &lt;transition on="success" to="assemblyFinalized"/&gt;
&lt;/action-state&gt;</pre>

<pre class="lang:java decode:true ">public String validateCarMechanics() {
    RequestContext context = RequestContextHolder.getRequestContext();
    Car car = (Car) context.getFlowScope().get("car");
    ...</pre>

&nbsp;

## <span style="color: #073763;">8 Accessing Spring beans</span>

If you need to retrieve beans from the Spring context, you can implement a utility class. The following method accesses the Spring context through the web flow request context.

<pre class="lang:java decode:true">public &lt;T&gt; T getBean(String name, Class&lt;T&gt; clazz) {
    RequestContext context = RequestContextHolder.getRequestContext();
    return context.getActiveFlow().getApplicationContext().getBean(name, clazz);
}

</pre>

&nbsp;

## <span style="color: #073763;">9 Using implicit variables</span>

There&#8217;s a full list of all available implicit variables at the web flow reference documentation

<a href="http://static.springsource.org/spring-webflow/docs/2.3.2.RELEASE/reference/htmlsingle/spring-webflow-reference.html#el-variables" target="_blank" rel="noopener">springsource web flow reference (el-variables)</a>