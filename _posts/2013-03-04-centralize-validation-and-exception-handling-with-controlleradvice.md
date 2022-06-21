---
id: 40
title: Centralize validation and exception handling with @ControllerAdvice
date: 2013-03-04T23:16:00+01:00
author: xpadro
layout: post
guid: http://xpadro.com/2013/03/04/centralize-validation-and-exception-handling-with-controlleradvice/
permalink: /2013/03/centralize-validation-and-exception-handling-with-controlleradvice.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2013/03/centralize-validation-and-exception.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/5876268825031348437
categories:
  - Spring
  - Spring MVC
tags:
  - MVC
  - REST
  - Spring
  - Test
---

The ControllerAdvice annotation introduced by Spring 3.2 allows us to handle several functionalities in a way that can be shared by all controllers (through its handler methods, annotated with @RequestMapping). This annotation is mainly used to define the following methods:

<ul>
  <li>
    <span style="font-family: 'Times New Roman','serif'; font-size: 12.0pt; mso-fareast-font-family: 'Times New Roman'; mso-fareast-language: ES;">@ExceptionHandler: Handles exceptions thrown by handler methods.</span>
  </li>
  <li>
    <span style="font-family: 'Times New Roman','serif'; font-size: 12.0pt; mso-fareast-font-family: 'Times New Roman'; mso-fareast-language: ES;">@InitBinder: Initializes the WebDataBinder, which will be used to populate objects passed as arguments to the handler methods.<b> </b>Usually, it is used to register property editors or validators.</span>
  </li>
  <li>
    <span style="font-family: 'Times New Roman','serif'; font-size: 12.0pt; mso-fareast-font-family: 'Times New Roman'; mso-fareast-language: ES;">@ModelAttribute: Binds a parameter or return value to an attribute, which will then be exposed to a web view.</span>
  </li>
</ul>

<div>
  Source code can be found at <a href="https://github.com/xpadro/spring-rest/tree/master/rest-controlleradvice" target="_blank" rel="noopener">github</a>.
</div>

<div>
</div>

<div>
  <h2>
    <span style="color: #073763;">1 Adding validation and exception handling</span>
  </h2>
</div>

<div>
  <span style="font-family: 'Times New Roman','serif'; font-size: 12.0pt; mso-fareast-font-family: 'Times New Roman'; mso-fareast-language: ES;">The following is a description of<b><i>  </i></b>the controller&#8217;s handler methods before implementing the @ControllerAdvice.</span>
</div>

<div>
  <b><i><span style="font-family: 'Times New Roman','serif'; font-size: 12.0pt; mso-fareast-font-family: 'Times New Roman'; mso-fareast-language: ES;">Add person controller:</span></i></b>
</div>


<div>
  <pre class="lang:java decode:true">@RequestMapping(value="/persons", method=RequestMethod.POST)
@ResponseStatus(HttpStatus.CREATED)
public void addPerson(@Valid @RequestBody Person person, HttpServletRequest request, HttpServletResponse response) {
    personRepository.addPerson(person);
    logger.info("Person added: "+person.getId());
    response.setHeader("Location", request.getRequestURL().append("/").append(person.getId()).toString());
}

@InitBinder
public voidinitBinder(WebDataBinder binder) {
    binder.setValidator(new PersonValidator());
}

@ExceptionHandler({MethodArgumentNotValidException.class})
publicResponseEntity&lt;String&gt; handleValidationException(MethodArgumentNotValidException pe) {
    return new ResponseEntity&lt;String&gt;(pe.getMessage(), HttpStatus.BAD_REQUEST);
}</pre>
  
  <p>
    &nbsp;
  </p>
</div>

<div>
  <span style="font-family: 'Times New Roman','serif'; font-size: 12.0pt; mso-fareast-font-family: 'Times New Roman'; mso-fareast-language: ES;">Besides the handler method, this controller has the following methods:</span>
</div>

<ul>
  <li>
    <span style="font-family: 'Times New Roman','serif'; font-size: 12.0pt; mso-fareast-font-family: 'Times New Roman'; mso-fareast-language: ES;">initBinder: Registers a validator to prevent that a person with invalid data is introduced. To make the validator validate the person object passed as a parameter, it is necessary to add the @Valid annotation to the argument. Spring 3 fully supports JSR-303 bean validation API, but it does not implement it. The reference implementation which is used in this example is Hibernate Validator 4.x.</span>
  </li>
  <li>
    <span style="font-family: 'Times New Roman','serif'; font-size: 12.0pt; mso-fareast-font-family: 'Times New Roman'; mso-fareast-language: ES;">handleValidationException: Handles the MethodArgumentNotValidException that can be thrown by the handler method. This exception is thrown by Spring MVC when an argument annotated with @Valid, fails its validation.</span>
  </li>
</ul>

<div>
  
  <div>
    <i><b>Get person controller:</b></i>
  </div>
  
  <div>
    <pre class="lang:java decode:true">@RequestMapping(value="/persons/{personId}", method=RequestMethod.GET)
public @ResponseBody Person getPerson(@PathVariable("personId") long id) {
    return personRepository.getPerson(id);
}

@ExceptionHandler({PersonNotFoundException.class})
publicResponseEntity&lt;String&gt; handlePersonNotFound(PersonNotFoundException pe) {
    return newResponseEntity&lt;String&gt;(pe.getMessage(), HttpStatus.NOT_FOUND);
}</pre>
    
    <p>
      &nbsp;
    </p>
  </div>
</div>

<div>
  <p>
    <span style="font-family: 'Times New Roman','serif'; font-size: 12.0pt; mso-fareast-font-family: 'Times New Roman'; mso-fareast-language: ES;">This controller adds an exception handler for handling when a request asks to retrieve a person that does not exist.</span>
  </p>
  
  <p>
    <span style="font-family: 'Times New Roman','serif'; font-size: 12.0pt; mso-fareast-font-family: 'Times New Roman'; mso-fareast-language: ES;"></span>
    
  </p>
  
  <div style="margin-bottom: 0cm;">
    <i><b><span lang="EN-US" style="mso-ansi-language: EN-US;">Update person controller:</span></b></i>
  </div>
  
  <div style="margin-bottom: 0cm;">
  </div>
  
  <div>
    <pre class="lang:java decode:true">@RequestMapping(value="/persons", method=RequestMethod.PUT)
@ResponseStatus(HttpStatus.CREATED)
public void updatePerson(@Valid @RequestBody Person person, HttpServletRequest request, HttpServletResponse response) {
    personRepository.updatePerson(person);
    logger.info("Person updated: "+person.getId());
    response.setHeader("Location", request.getRequestURL().append("/").append(person.getId()).toString());
}

@InitBinder
public void initBinder(WebDataBinder binder) {
    binder.setValidator(new PersonValidator());
}

@ExceptionHandler({PersonNotFoundException.class})
public ResponseEntity&lt;String&gt; handlePersonNotFound(PersonNotFoundException pe) {
    return newResponseEntity&lt;String&gt;(pe.getMessage(), HttpStatus.NOT_FOUND);
}

@ExceptionHandler({Exception.class})
public ResponseEntity&lt;String&gt; handleValidationException(MethodArgumentNotValidException pe) {
    return newResponseEntity&lt;String&gt;(pe.getMessage(), HttpStatus.BAD_REQUEST);
}</pre>
    
    <p>
      &nbsp;
    </p>
  </div>
  
  <p>

    <span style="font-family: 'Times New Roman','serif'; font-size: 12.0pt; mso-fareast-font-family: 'Times New Roman'; mso-fareast-language: ES;"><span style="font-family: Consolas;">W</span>e are repeating code, since @ExceptionHandler is not global.</span>
  </p>
</div>

## <span style="color: #073763;">2 Centralizing code</span>

ControllerAdvice annotation is itself annotated with @Component. Hence, the class that we are implementing will be autodetected through classpath scanning.

<div>
  
  <div>
    <pre class="lang:java decode:true">@ControllerAdvice
public classCentralControllerHandler {
@InitBinder
public void initBinder(WebDataBinder binder) {
    binder.setValidator(new PersonValidator());
}

@ExceptionHandler({PersonNotFoundException.class})
public ResponseEntity&lt;String&gt; handlePersonNotFound(PersonNotFoundException pe) {
    return newResponseEntity&lt;String&gt;(pe.getMessage(), HttpStatus.NOT_FOUND);
}

@ExceptionHandler({MethodArgumentNotValidException.class})
public ResponseEntity&lt;String&gt; handleValidationException(MethodArgumentNotValidException pe) {
    return newResponseEntity&lt;String&gt;(pe.getMessage(), HttpStatus.BAD_REQUEST);
}
}</pre>
    
    <p>
      <span style="font-family: 'Times New Roman', serif; font-size: 12pt;">Finally, we can delete these methods from the controllers, taking rid of code duplication, since this class will handle exception handling and validation for all handler methods annotated with @RequestMapping.</span>
    </p>
  </div>
</div>

<div>
  <span style="font-family: 'Times New Roman','serif'; font-size: 12.0pt; mso-fareast-font-family: 'Times New Roman'; mso-fareast-language: ES;">  </span>
</div>

## <span style="color: #073763;">3 Testing</span>

The methods described below, test the retrieval of persons:

<div>
  <pre class="lang:java decode:true ">@Test
public void getExistingPerson() {
    String uri = "http://localhost:8081/rest-controlleradvice/spring/persons/{personId}";
    Person person = restTemplate.getForObject(uri, Person.class, 1l);
    assertNotNull(person);
    assertEquals("Xavi", person.getName());
}

@Test
public void getNonExistingPerson() {
    String uri = "http://localhost:8081/rest-controlleradvice/spring/persons/{personId}";
    try {
        restTemplate.getForObject(uri, Person.class, 5l);
        throw new AssertionError("Should have returned an 404 error code");
    } catch(HttpClientErrorException e) {
        assertEquals(HttpStatus.NOT_FOUND, e.getStatusCode());
    }
}</pre>
  
  <p>
    &nbsp;
  </p>
</div>

<div>
  <span style="font-family: 'Times New Roman','serif'; font-size: 12.0pt; mso-fareast-font-family: 'Times New Roman'; mso-fareast-language: ES;">The rest of tests can be found with the source code linked above.</span>
</div>
