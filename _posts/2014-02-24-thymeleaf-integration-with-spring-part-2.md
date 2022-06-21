---
id: 26
title: Thymeleaf integration with Spring (Part 2)
date: 2014-02-24T13:04:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2014/02/24/thymeleaf-integration-with-spring-part-2/
permalink: /2014/02/thymeleaf-integration-with-spring-part-2.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2014/02/thymeleaf-integration-with-spring-part-2.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/2786001648464880466
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

This is the second part of the Thymeleaf integration with Spring tutorial. You can read the first part <a href="http://xpadro.com/2014/02/thymeleaf-integration-with-spring-part-1.html" target="_blank" rel="noopener">here</a>, where you will learn how to configure this project.

As explained at the beginning of the first part of this tutorial, the web application will send two types of requests:

  * Insert a new guest: Sends a synchronous request to the server for adding a new guest. This will demonstrate how Thymeleaf is integrated with Spring’s form backing beans.

  * List guests: Sends an AJAX request that will update a region (<a href="http://www.thymeleaf.org/doc/html/Using-Thymeleaf.html#including-template-fragments" target="_blank" rel="noopener">fragment</a>) of the page in order to show the guest list returned from the server.

Let&#8217;s see how we will accomplish this.

## <span style="color: #0b5394;">1 Handling forms</span>

In this section we are going to see how a form is submitted with Thymeleaf. We will basically need three attributes:

th:action  
th:object  
th:field

The first two are defined in the form element:

<pre class="lang:xhtml decode:true ">&lt;form id="guestForm" th:action="@{/spring/guests/insert}" th:object="${guest}" method="post"&gt;</pre>

&nbsp;

The _th:action_ attribute rewrites the action url, prefixing the application context to it.

The _th:object_ attribute in the form element is the object selection. It can then be referenced within the form. What we do here is bind the form backing bean to the model attribute, which we defined in the controller before rendering the view:

<pre class="lang:java decode:true ">@ModelAttribute("guest")
public Guest prepareGuestModel() {
    return new Guest();
}</pre>

&nbsp;

As we see, _th:object_ refers to the Guest form backing bean, while _th:field_ will refer to its properties.  Take a look at the form body:

<pre class="lang:xhtml decode:true ">&lt;span class="formSpan"&gt;
    &lt;input id="guestId" type="text" th:field="*{id}" required="required"/&gt;
    &lt;br /&gt;
    &lt;label for="guestId" th:text="#{insert.id}"&gt;id:&lt;/label&gt;
&lt;/span&gt;

&lt;span class="formSpan" style="margin-bottom:20px"&gt;
    &lt;input id="guestName" type="text" th:field="*{name}" required="required"/&gt;
    &lt;br /&gt;
    &lt;label for="guestName" th:text="#{insert.name}"&gt;name:&lt;/label&gt;
&lt;/span&gt;</pre>

&nbsp;

What _th:field_ will do is assign the value of its input element to the backing bean property. So, when the user submits the form, all these _th:field_ will set the form backing bean properties.

At the controller, we will receive the Guest instance:

<pre class="lang:java decode:true ">@RequestMapping(value = "/guests/insert", method = RequestMethod.POST)
public String insertGuest(Guest newGuest, Model model) {
    hotelService.insertNewGuest(newGuest);
    
    return showHome(model);
}</pre>

&nbsp;

Now the guest can be inserted into the database.

## <span style="color: #0b5394;">2 Sending AJAX requests</span>

When trying to find a simple example of sending an AJAX request with Thymeleaf, I have found examples with Spring Webflow (<a href="https://github.com/spring-projects/spring-webflow-samples/blob/master/booking-mvc/src/main/webapp/WEB-INF/hotels/booking/booking-flow.xml" target="_blank" rel="noopener">render fragments</a>). I also <a href="http://stackoverflow.com/questions/15739584/ajax-in-the-spring-mvc-thymeleaf-application" target="_blank" rel="noopener">read </a>others saying that you need Tiles in order to accomplish that.

I didn&#8217;t want to use those frameworks so in this section I&#8217;m using jQuery to send an AJAX request to the server, wait for the response and partially update the view (fragment rendering).

#### <span style="color: #003366;">The form</span>

<pre class="lang:xhtml decode:true">&lt;form&gt;
    &lt;span class="subtitle"&gt;Guest list form&lt;/span&gt;
    &lt;div class="listBlock"&gt;
        &lt;div class="search-block"&gt;
            &lt;input type="text" id="searchSurname" name="searchSurname"/&gt;
            &lt;br /&gt;
            &lt;label for="searchSurname" th:text="#{search.label}"&gt;Search label:&lt;/label&gt;
            &lt;button id="searchButton" name="searchButton" onclick="retrieveGuests()" type="button" 
                    th:text="#{search.button}"&gt;Search button&lt;/button&gt;
        &lt;/div&gt;
        
        &lt;!-- Results block --&gt;
        &lt;div id="resultsBlock"&gt;
        
        &lt;/div&gt;
    &lt;/div&gt;
&lt;/form&gt;</pre>

This form contains an input text with a search string (searchSurname) that will be sent to the server. There&#8217;s also a region (resultsBlock div) which will be updated with the response received from the server.

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://3.bp.blogspot.com/-FjJyjOonu3k/VUZZ49OG6OI/AAAAAAAAEp4/OPFaB4PTGx4/s1600/searchForm.png"><img loading="lazy" class="alignnone" src="https://3.bp.blogspot.com/-FjJyjOonu3k/VUZZ49OG6OI/AAAAAAAAEp4/OPFaB4PTGx4/s1600/searchForm.png" alt="Guests list form" width="282" height="90" border="0" /></a>
</div>

&nbsp;

When the user clicks the button, the retrieveGuests() function will be invoked.

<pre class="lang:js decode:true ">function retrieveGuests() {
    var url = '/th-spring-integration/spring/guests';
    
    if ($('#searchSurname').val() != '') {
        url = url + '/' + $('#searchSurname').val();
    }
    
    $("#resultsBlock").load(url);
}</pre>

&nbsp;

The jQuery <a href="https://api.jquery.com/load/" target="_blank" rel="noopener">load</a> function makes a request to the server at the specified url and places the returned HTML into the specified element (resultsBlock div).

If the user enters a search string, it will search for all guests with the specified surname. Otherwise, it will return the complete guest list. These two requests will reach the following controller request mappings:

<pre class="lang:java decode:true ">@RequestMapping(value = "/guests/{surname}", method = RequestMethod.GET)
public String showGuestList(Model model, @PathVariable("surname") String surname) {
    model.addAttribute("guests", hotelService.getGuestsList(surname));
    
    return "results :: resultsList";
}

@RequestMapping(value = "/guests", method = RequestMethod.GET)
public String showGuestList(Model model) {
    model.addAttribute("guests", hotelService.getGuestsList());
    
    return "results :: resultsList";
}</pre>

&nbsp;

#### <span style="color: #003366;">The results list</span>

Since Spring is integrated with Thymeleaf, it will now be able to return fragments of HTML. In the above example, the return string &#8220;results :: resultsList&#8221; is referring to a fragment named resultsList which is located in the results page. Let&#8217;s take a look at this results page:

<pre class="lang:xhtml decode:true ">&lt;!DOCTYPE html&gt;
&lt;html xmlns="http://www.w3.org/1999/xhtml"
    xmlns:th="http://www.thymeleaf.org" lang="en"&gt;

&lt;head&gt;
&lt;/head&gt;

&lt;body&gt;
    &lt;div th:fragment="resultsList" th:unless="${#lists.isEmpty(guests)}" class="results-block"&gt;
        &lt;table&gt;
            &lt;thead&gt;
                &lt;tr&gt;
                    &lt;th th:text="#{results.guest.id}"&gt;Id&lt;/th&gt;
                    &lt;th th:text="#{results.guest.surname}"&gt;Surname&lt;/th&gt;
                    &lt;th th:text="#{results.guest.name}"&gt;Name&lt;/th&gt;
                    &lt;th th:text="#{results.guest.country}"&gt;Country&lt;/th&gt;
                &lt;/tr&gt;
            &lt;/thead&gt;
            &lt;tbody&gt;
                &lt;tr th:each="guest : ${guests}"&gt;
                    &lt;td th:text="${guest.id}"&gt;id&lt;/td&gt;
                    &lt;td th:text="${guest.surname}"&gt;surname&lt;/td&gt;
                    &lt;td th:text="${guest.name}"&gt;name&lt;/td&gt;
                    &lt;td th:text="${guest.country}"&gt;country&lt;/td&gt;
                &lt;/tr&gt;
            &lt;/tbody&gt;
        &lt;/table&gt;
    &lt;/div&gt;
&lt;/body&gt;
&lt;/html&gt;</pre>

&nbsp;

The fragment, which is a table with registered guests, will be inserted in the results block:

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://2.bp.blogspot.com/-_7L3SPl1fdA/VUZZ4xaJU-I/AAAAAAAAEp0/DXyuvOln1SA/s1600/resultsBlock.png"><img loading="lazy" class="alignnone" src="https://2.bp.blogspot.com/-_7L3SPl1fdA/VUZZ4xaJU-I/AAAAAAAAEp0/DXyuvOln1SA/s1600/resultsBlock.png" alt="Thymeleaf integration with Spring - Part 2. Results page" width="400" height="170" border="0" /></a>
</div>

&nbsp;

## <span style="color: #0b5394;">3 Conclusion</span>

After integrating both frameworks, we learnt how forms are linked to the Spring MVC model. We also learnt how to send AJAX requests and partially update the view.

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.

&nbsp;