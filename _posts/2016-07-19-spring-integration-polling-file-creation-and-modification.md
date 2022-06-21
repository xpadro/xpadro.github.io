---
id: 8
title: 'Spring Integration &#8211; Polling file creation and modification'
date: 2016-07-19T12:48:00+01:00
author: xpadro
layout: post
permalink: /2016/07/spring-integration-polling-file-creation-and-modification.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2016/07/spring-integration-polling-file.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/122859828559349491
categories:
  - Integration
  - Spring
  - Spring-Boot
tags:
  - Integration
  - Spring
---

File support is another of Spring Integration&#8217;s endpoints to communicate with external systems. In this case, it provides several components to read, write and transform files. During this post, we are going to write an application which monitors a directory in order to read all files in there. In concrete it does the following:

  * When the application starts, it reads all files present in the directory.
  * The application will then keep an eye on the directory to detect new files and existing files which have been modified.

The source code can be found in [Github](https://github.com/xpadro/spring-integration/tree/master/file).

## <span style="color: #0b5394;">1 Configuration</span>

The application is built with Spring Boot, since it eases configuration significantly. To create the initial infrastructure of the application, you can go to <https://start.spring.io/>, select the Integration module and generate the project. Then you can open the zip file in your favourite IDE.

I added a couple of dependencies to the pom.xml like commons.io or Spring Integration Java DSL. My pom.xml file looks like as follows:

<pre class="lang:xhtml decode:true ">&lt;?xml version="1.0" encoding="UTF-8"?&gt;
&lt;project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd"&gt;
  &lt;modelVersion&gt;4.0.0&lt;/modelVersion&gt;

  &lt;groupId&gt;xpadro.spring.integration&lt;/groupId&gt;
  &lt;artifactId&gt;file-read-directory&lt;/artifactId&gt;
  &lt;version&gt;0.0.1-SNAPSHOT&lt;/version&gt;
  &lt;packaging&gt;jar&lt;/packaging&gt;

  &lt;name&gt;file-read-directory&lt;/name&gt;
  &lt;description&gt;Demo project for Spring Boot&lt;/description&gt;

  &lt;parent&gt;
    &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
    &lt;artifactId&gt;spring-boot-starter-parent&lt;/artifactId&gt;
    &lt;version&gt;1.3.5.RELEASE&lt;/version&gt;
    &lt;relativePath/&gt; &lt;!-- lookup parent from repository --&gt;
  &lt;/parent&gt;

  &lt;properties&gt;
    &lt;project.build.sourceEncoding&gt;UTF-8&lt;/project.build.sourceEncoding&gt;
    &lt;java.version&gt;1.8&lt;/java.version&gt;
  &lt;/properties&gt;

  &lt;dependencies&gt;
    &lt;dependency&gt;
      &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
      &lt;artifactId&gt;spring-boot-starter-integration&lt;/artifactId&gt;
    &lt;/dependency&gt;

    &lt;dependency&gt;
      &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
      &lt;artifactId&gt;spring-boot-starter-test&lt;/artifactId&gt;
      &lt;scope&gt;test&lt;/scope&gt;
    &lt;/dependency&gt;

    &lt;!-- Spring Integration - Java DSL --&gt;
    &lt;dependency&gt;
      &lt;groupId&gt;org.springframework.integration&lt;/groupId&gt;
      &lt;artifactId&gt;spring-integration-java-dsl&lt;/artifactId&gt;
      &lt;version&gt;1.0.0.RELEASE&lt;/version&gt;
    &lt;/dependency&gt;

    &lt;dependency&gt;
      &lt;groupId&gt;commons-io&lt;/groupId&gt;
      &lt;artifactId&gt;commons-io&lt;/artifactId&gt;
      &lt;version&gt;2.5&lt;/version&gt;
    &lt;/dependency&gt;

  &lt;/dependencies&gt;

  &lt;build&gt;
    &lt;plugins&gt;
      &lt;plugin&gt;
        &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
        &lt;artifactId&gt;spring-boot-maven-plugin&lt;/artifactId&gt;
      &lt;/plugin&gt;
    &lt;/plugins&gt;
  &lt;/build&gt;

&lt;/project&gt;</pre>

&nbsp;

The starting point is FileReadDirectoryApplication:

<pre class="lang:java decode:true ">@SpringBootApplication
public class FileReadDirectoryApplication {

    public static void main(String[] args) throws IOException, InterruptedException {
		SpringApplication.run(FileReadDirectoryApplication.class, args);
	}
}</pre>

&nbsp;

Starting from here, we are going to add the Spring Integration components for reading from a specific folder of the filesystem.

## <span style="color: #0b5394;">2 Adding the adapter</span>

In order to read from the file system, we need an inbound channel adapter. The adapter is a file reading message source, which is responsible of polling the file system directory for files and create a message from each file it finds.

<pre class="lang:java decode:true ">@Bean
@InboundChannelAdapter(value = "fileInputChannel", poller = @Poller(fixedDelay = "1000"))
public MessageSource&lt;File&gt; fileReadingMessageSource() {
    CompositeFileListFilter&lt;File&gt; filters = new CompositeFileListFilter&lt;&gt;();
    filters.addFilter(new SimplePatternFileListFilter("*.txt"));
    filters.addFilter(new LastModifiedFileFilter());
    
    FileReadingMessageSource source = new FileReadingMessageSource();
    source.setAutoCreateDirectory(true);
    source.setDirectory(new File(DIRECTORY));
    source.setFilter(filters);
    
    return source;
}</pre>

&nbsp;

We can prevent some types of files from being polled by setting a list of filters to the message source. For this example two filters have been included:

  * _SimplePatternFileListFilter_: Filter provided by Spring. Only files with the specified extension will be polled. In this case, only text files will be accepted.
  * _LastModifiedFileFilter_: Custom filter. This filter keeps track of already polled files and will filter out files not modified since the last time it was tracked.

&nbsp;

## <span style="color: #0b5394;">3 Processing the files</span>

For each polled file, we will transform its content to String before passing it to the processor. For this purpose, Spring already provides a component:

<pre class="lang:java decode:true ">@Bean
public FileToStringTransformer fileToStringTransformer() {
    return new FileToStringTransformer();
}</pre>

&nbsp;

Hence, instead of receiving a Message<File>, the processor will receive a Message<String>. The file processor is our custom component which will do something as advanced as printing the file content:

<pre class="lang:java decode:true ">public class FileProcessor {
    private static final String HEADER_FILE_NAME = "file_name";
    private static final String MSG = "%s received. Content: %s";
    
    public void process(Message&lt;String&gt; msg) {
        String fileName = (String) msg.getHeaders().get(HEADER_FILE_NAME);
        String content = msg.getPayload();
        
        System.out.println(String.format(MSG, fileName, content));
    }
}</pre>

&nbsp;

## <span style="color: #0b5394;">4 Building the flow</span>

Now that we have all the required components in place, let&#8217;s build the flow. We are using Spring Integration Java DSL, since it makes the flow more readable:

<pre class="lang:java decode:true ">@Bean
public IntegrationFlow processFileFlow() {
    return IntegrationFlows
        .from("fileInputChannel")
        .transform(fileToStringTransformer())
        .handle("fileProcessor", "process").get();
    }
    
    @Bean
    public MessageChannel fileInputChannel() {
        return new DirectChannel();
    }</pre>

&nbsp;

## <span style="color: #0b5394;">5 Running the application</span>

In my directory, I already have a file called &#8216;previousFile.txt&#8217;. After starting the application, we will create two files and modify one of them.

<pre class="lang:java decode:true ">public static void main(String[] args) throws IOException, InterruptedException {
    SpringApplication.run(FileReadDirectoryApplication.class, args);
    createFiles();
}

private static void createFiles() throws IOException, InterruptedException {
    createFile("file1.txt", "content");
    createFile("file2.txt", "another file");
    appendFile("file1.txt", " modified");
}</pre>

&nbsp;

If we run the application, we should see the following print statements:

<span style="font-size: x-small;">previousFile.txt received. Content: previous content</span>  
<span style="font-size: x-small;">file1.txt received. Content: content</span>  
<span style="font-size: x-small;">file2.txt received. Content: another file</span>  
<span style="font-size: x-small;">file1.txt received. Content: content modified</span>

## <span style="color: #0b5394;">6 Conclusion</span>

This example shows how simple it is to read files from a directory using Spring Integration, obviously with the help of Spring Boot to simplify the configuration. Depending on your needs, you can add your own custom filters to the message source, or use another one of the provided by Spring, like the [RegexPatternFileListFilter](http://docs.spring.io/spring-integration/api/org/springframework/integration/file/filters/RegexPatternFileListFilter.html). You can check for other implementations [here](http://docs.spring.io/spring-integration/api/org/springframework/integration/file/filters/FileListFilter.html).

If you found this post useful, please share it or star my repository 🙂

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.

&nbsp;