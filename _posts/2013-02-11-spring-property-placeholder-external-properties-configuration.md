---
id: 42
title: 'Spring property-placeholder: External properties configuration'
date: 2013-02-11T20:48:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/02/11/spring-property-placeholder-external-properties-configuration/
permalink: /2013/02/spring-property-placeholder-external-properties-configuration.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/02/spring-property-placeholder-external.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/8532388883952091019
categories:
  - Spring
tags:
  - Spring
---

A common way of setting the configuration of the web application is by defining it in a properties file. We can find this file in the classpath. The way of doing this is using the PropertyPlaceholderConfigurer class, or simplifying the configuration, using the context namespace property-placeholder.

<pre class="lang:xhtml decode:true">&lt;context:property-placeholder location="datasource.properties"/&gt;</pre>

That&#8217;s ok but, what if you need a different configuration for different environments? Or, once the web application is deployed on the server, what happens if we need to change some value? What we will have to do is modify the application, redeploy the war file and restart the server. If there&#8217;s only one application deployed, then it&#8217;s not much of a problem, but imagine a properties file which contains generic configuration that is shared by all the applications on the server.

## <span style="color: #0b5394;">Externalizing the properties file </span>

By putting the properties file in the server file system, you will have your configuration outside the web application. In case you need to change a value, you will only need to modify the file and restart the server, without needing to redeploy the application.

Let&#8217;s assume that we put our properties file in the &#8220;D:environmentconfig&#8221; folder. What we need to do is add a new argument to the server start up script. If you use an IDE, for example Eclipse, double click the server name (servers tab) and click on the Open launch configuration link:

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://3.bp.blogspot.com/-GfrE2wCe6bg/VUY7JDVbBCI/AAAAAAAAEjQ/M5zBOVvety4/s1600/serverOverview.png"><img loading="lazy" class="alignnone" src="http://3.bp.blogspot.com/-GfrE2wCe6bg/VUY7JDVbBCI/AAAAAAAAEjQ/M5zBOVvety4/s1600/serverOverview.png" alt="property-placeholder-launch-configuration" width="320" height="285" border="0" /></a>
</div>

Then, select the arguments tab and add a new argument with the properties path as shown below:

<div style="clear: both; text-align: center;">
</div>

<div style="clear: both; text-align: center;">
  <a style="margin-left: 1em; margin-right: 1em;" href="http://1.bp.blogspot.com/-Wr0O8l9lch4/VUY7KjQVN4I/AAAAAAAAEjY/5kXrzEg4wF4/s1600/serverArguments.png"><img loading="lazy" class="alignnone" src="http://1.bp.blogspot.com/-Wr0O8l9lch4/VUY7KjQVN4I/AAAAAAAAEjY/5kXrzEg4wF4/s1600/serverArguments.png" alt="vm arguments" width="640" height="224" border="0" /></a>
</div>

Once we set the arguments, the Spring configuration with the property placeholder is as follows:

<pre class="lang:xhtml decode:true ">&lt;context:property-placeholder location="file:${databaseConfiguration}"/&gt;

&lt;bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource"&gt;
    &lt;property name="url" value="${jdbc.url}" /&gt;
    &lt;property name="username" value="${jdbc.username}" /&gt;
    &lt;property name="password" value="${jdbc.password}" /&gt;
&lt;/bean&gt;</pre>

&nbsp;