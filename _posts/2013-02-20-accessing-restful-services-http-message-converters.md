---
id: 41
title: Accessing Restful services. HTTP Message converters
date: 2013-02-20T14:06:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/02/20/accessing-restful-services-http-message-converters/
permalink: /2013/02/accessing-restful-services-http-message-converters.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/02/accessing-restful-services-http-message.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/2339292740882010955
categories:
  - Spring
  - Spring MVC
tags:
  - MVC
  - REST
  - Spring
---

When accessing to Restful services, the Spring class RestTemplate maintains a list of message converters. This list will be used to marshal objects into the request body, or unmarshalling them from the response. When instantiating the RestTemplate class, it automatically fills a list with several converters:

  * <span lang="EN-US">ByteArrayHttpMessageConverter</span>
  * <span lang="EN-US">StringHttpMessageConverter</span>
  * <span lang="EN-US">ResourceHttpMessageConverter</span>
  * <span lang="EN-US">SourceHttpMessageConverter</span>
  * <span lang="EN-US">AllEncompassingFormHttpMessageConverter</span>

<div>
   <span lang="EN-US" style="mso-ansi-language: EN-US;"><br /> </span><span lang="EN-US" style="mso-ansi-language: EN-US;">Also, the RestTemplate class may register additional message converters if it finds the required classes on the classpath:</span>
</div>

  * <span lang="EN-US" style="font-family: Symbol; mso-ansi-language: EN-US; mso-bidi-font-family: Symbol; mso-fareast-font-family: Symbol;"><span style="font: 7.0pt 'Times New Roman';"> </span></span><span lang="EN-US" style="mso-ansi-language: EN-US;">JAXB converter: Converts xml using JAXB2. The Binder class must be found on the classpath. If you are using Java 6, it is no longer necessary as it already comes with the JDK. If you are using Java 5, add the following dependencies:</span>

<div>
  <pre class="lang:xhtml decode:true">&lt;dependency&gt;
    &lt;groupId&gt;javax.xml.bind&lt;/groupId&gt;
    &lt;artifactId&gt;jaxb-api&lt;/artifactId&gt;
    &lt;version&gt;2.2&lt;/version&gt;
&lt;/dependency&gt;
&lt;dependency&gt;
    &lt;groupId&gt;com.sun.xml.bind&lt;/groupId&gt;
    &lt;artifactId&gt;jaxb-impl&lt;/artifactId&gt;
    &lt;version&gt;2.2&lt;/version&gt;
&lt;/dependency&gt;</pre>
</div>

&nbsp;

  * <span style="text-indent: -18pt;">Jackson converter: Converts Objects from/to JSON. ObjectMapper and JsonGenerator must exist on the classpath.</span>

<div>
  <pre class="lang:xhtml decode:true">&lt;dependency&gt;
                  &lt;groupId&gt;org.codehaus.jackson&lt;/groupId&gt;
                  &lt;artifactId&gt;jackson-mapper-asl&lt;/artifactId&gt;
                  &lt;version&gt;1.4.2&lt;/version&gt;
            &lt;/dependency&gt;</pre>
  
  <p>
    &nbsp;
  </p>
</div>

  * <span lang="EN-US" style="font-family: Symbol; mso-ansi-language: EN-US; mso-bidi-font-family: Symbol; mso-fareast-font-family: Symbol;"><span style="font: 7.0pt 'Times New Roman';"> </span></span><span lang="EN-US" style="mso-ansi-language: EN-US;">Atom and RSS feeds converters. The WireFeed class must be present on the classpath.</span>


## <span style="color: #073763;"><span lang="EN-US" style="mso-ansi-language: EN-US;">Serving different content</span></span>

<div>
   <span lang="EN-US" style="mso-ansi-language: EN-US;">The following controller will execute three different operations that will generate xml, String and json content.</span>
</div>

<div>
  <strong>JSON content:</strong>
</div>

<div>
  <pre class="lang:java decode:true">@RequestMapping(value="/users/{userId}", method=RequestMethod.GET)
public @ResponseBody User getUser(@PathVariable("userId") long id) {
       return userRepository.getUser(id);
}</pre>
  
  <p>
    <span style="color: black; font-family: Consolas; font-size: 10pt;">Result: JSON. The User class can be serialized/deserialized by the Jackson object mapper.</span>
  </p>
</div>

<div>
  <pre class="lang:js decode:true">{
name: "Xavi",
id: 1,
surname: "Padro"
}</pre>
</div>

<div>
  String content:
</div>

<div>
  <pre class="lang:java decode:true ">@RequestMapping(value="/usernames/{userId}", method=RequestMethod.GET)
public @ResponseBody String getUsername(@PathVariable("userId") long id) {
       StringBuilder result = new StringBuilder();
       User user = userRepository.getUser(id);
       returnresult.append(user.getName()).append(" ").append(user.getSurname()).toString();
}</pre>
  
  <p>
    <span style="color: black; font-family: Consolas; font-size: 10pt;">Result: String. The returned object is a String.</span>
  </p>
</div>

<div>
  <div>
    Xavi Padro
  </div>
</div>

<div>
  XML content:
</div>

<div>
  <pre class="lang:java decode:true">@RequestMapping(value="/cars/{carId}", method=RequestMethod.GET)
public @ResponseBody Car getCar(@PathVariable("carId") long id) {
       return carRepository.getCar(id);
}</pre>
  
  <p>
    &nbsp;
  </p>
</div>

<div>
  <span style="color: black; font-family: Consolas; font-size: 10.0pt; line-height: 115%;">Result: XML. The Car class is annotated with @XmlRootElement</span>
</div>

<div>
  <pre class="lang:xhtml decode:true ">&lt;car&gt;
&lt;brand&gt;Ferrari&lt;/brand&gt;
&lt;id&gt;1&lt;/id&gt;
&lt;model&gt;GTO&lt;/model&gt;
&lt;/car&gt;</pre>
</div>

<div>
  <span lang="EN-US" style="mso-ansi-language: EN-US;">You can also specify the converters that will be used by the RestTemplate. You could, for example, only instantiate the needed converters or register your own message converter:</span>
</div>

<div>
  <pre class="lang:java decode:true">private RestTemplate restTemplate = new RestTemplate();

@Before
public void setup() {
List&lt;HttpMessageConverter&lt;?&gt;&gt; converters = new ArrayList&lt;HttpMessageConverter&lt;?&gt;&gt;();
       converters.add(newStringHttpMessageConverter());
       converters.add(newJaxb2RootElementHttpMessageConverter());
       converters.add(newMappingJacksonHttpMessageConverter());
       restTemplate.setMessageConverters(converters);
}</pre>
  
  <p>
    &nbsp;
  </p>
</div>

<div>
  <span lang="EN-US" style="mso-ansi-language: EN-US;">Just take into account that if you serve both json and xml content, you must add the jaxb2 converter before the json converter. Otherwise, regardless of the presence of xml annotations, the xml method response will be handled by the jackson converter and be converted to json. That happens because the HttpEntityRequestCallback inner class of the RestTemplate will use the first converter that can write/read the content.</span>
</div>

&nbsp;

<div>
  You can get the source code from <a href="https://github.com/xpadro/spring-rest" target="_blank" rel="noopener">Github repository</a>.
</div>