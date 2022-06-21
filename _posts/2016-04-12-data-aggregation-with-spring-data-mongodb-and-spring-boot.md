---
id: 10
title: Data aggregation with Spring Data MongoDB and Spring Boot
date: 2016-04-12T13:13:00+01:00
author: xpadro
layout: post
permalink: /2016/04/data-aggregation-with-spring-data-mongodb-and-spring-boot.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2016/04/data-aggregation-with-spring-data.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/6552777420412304289
image: /wp-content/uploads/2016/04/aggregationPipeline.png
categories:
  - mongoDB
  - Spring
  - Spring-Boot
  - Spring-Data
tags:
  - MongoDB
  - Spring
  - Spring Boot
  - Test
---

MongoDB aggregation framework is designed for grouping documents and transforming them into an aggregated result. The aggregation query consists in defining several stages that will be executed in a pipeline. If you are interested in more in-depth details about the framework, then <a href="https://docs.mongodb.org/manual/aggregation/" target="_blank" rel="noopener">mongodb docs</a> are a good point to start.

The point of this post is to write a web application for querying mongodb in order to get aggregated results from the database. We will do it in a very easy way thanks to Spring Boot and Spring Data. Actually it is really fast to implement the application, since Spring Boot will take care of all the necessary setup and Spring Data will help us configure the repositories.

The source code can be found on my [Github repository](https://github.com/xpadro/spring-data-mongo).

## <span style="color: #0b5394;">1 The application</span>

Before going through the code let’s see what we want to do with our application.

Our domain is a collection of products we have distributed across several warehouses:

<pre class="lang:java decode:true ">@Document
public class Product {
    
    @Id
    private final String id;
    private final String warehouse;
    private final float price;
    
    public Product(String id, String warehouse, float price) {
        this.id = id;
        this.warehouse = warehouse;
        this.price = price;
    }
    
    public String getId() {
        return id;
    }
    
    public String getWarehouse() {
        return warehouse;
    }
    
    public float getPrice() {
        return price;
    }
}</pre>

&nbsp;

Our target is to collect all the products within a price range, grouped by warehouse and collecting the total revenue and the average price of each grouping.

In this example, our warehouses are storing the following products:

<pre class="lang:java decode:true ">new Product("NW1", "Norwich", 3.0f);
new Product("LN1", "London", 25.0f);
new Product("LN2", "London", 35.0f);
new Product("LV1", "Liverpool", 15.2f);
new Product("MN1", "Manchester", 45.5f);
new Product("LV2", "Liverpool", 23.9f);
new Product("LN3", "London", 55.5f);
new Product("LD1", "Leeds", 87.0f);</pre>

&nbsp;

The application will query for products with a price between 5.0 and 70.0. The required aggregation pipeline steps will be as follows:

<div style="clear: both; text-align: center;">
  <img loading="lazy" class="alignnone wp-image-245 size-full" src="http://xpadro.com/wp-content/uploads/2016/04/aggregationPipeline_optimized.png" alt="MongoDB data aggregation using Spring - pipeline" width="768" height="506" srcset="https://xpadro.com/wp-content/uploads/2016/04/aggregationPipeline_optimized.png 768w, https://xpadro.com/wp-content/uploads/2016/04/aggregationPipeline_optimized-300x198.png 300w, https://xpadro.com/wp-content/uploads/2016/04/aggregationPipeline_optimized-320x211.png 320w, https://xpadro.com/wp-content/uploads/2016/04/aggregationPipeline_optimized-640x422.png 640w, https://xpadro.com/wp-content/uploads/2016/04/aggregationPipeline_optimized-360x237.png 360w, https://xpadro.com/wp-content/uploads/2016/04/aggregationPipeline_optimized-720x474.png 720w" sizes="(max-width: 768px) 100vw, 768px" />
</div>

&nbsp;

We will end up with aggregated results grouped by warehouse. Each group will contain the list of products of each warehouse, the average product price and the total revenue, which actually is the sum of the prices.

## <span style="color: #0b5394;">2 Maven dependencies</span>

As you can see, we have a short <a href="https://github.com/xpadro/spring-data-mongo/blob/master/aggregation-basic/pom.xml" target="_blank" rel="noopener">pom.xml</a> with Spring Boot dependencies:

<pre class="lang:xhtml decode:true ">&lt;parent&gt;
    &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
    &lt;artifactId&gt;spring-boot-starter-parent&lt;/artifactId&gt;
    &lt;version&gt;1.3.3.RELEASE&lt;/version&gt;
    &lt;relativePath/&gt;
&lt;/parent&gt;

&lt;properties&gt;
    &lt;project.build.sourceEncoding&gt;UTF-8&lt;/project.build.sourceEncoding&gt;
    &lt;java.version&gt;1.8&lt;/java.version&gt;
&lt;/properties&gt;

&lt;dependencies&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
        &lt;artifactId&gt;spring-boot-starter-data-mongodb&lt;/artifactId&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
        &lt;artifactId&gt;spring-boot-starter-web&lt;/artifactId&gt;
    &lt;/dependency&gt;
    
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
        &lt;artifactId&gt;spring-boot-starter-test&lt;/artifactId&gt;
        &lt;scope&gt;test&lt;/scope&gt;
    &lt;/dependency&gt;
&lt;/dependencies&gt;

&lt;build&gt;
    &lt;plugins&gt;
        &lt;plugin&gt;
            &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
            &lt;artifactId&gt;spring-boot-maven-plugin&lt;/artifactId&gt;
        &lt;/plugin&gt;
    &lt;/plugins&gt;
&lt;/build&gt;</pre>

&nbsp;

By defining spring-boot-starter-parent as our parent pom, we set the default settings of Spring Boot. Mainly it sets the versions of a bunch of libraries it may use, like Spring or Apache Commons. For example, Spring Boot 1.3.3, which is the one we are using, sets 4.2.5.RELEASE as the Spring framework version. Like stated in previous posts, it is not adding libraries to our application, it only sets versions.

Once the parent is defined, we only need to add three dependencies:

  * spring-boot-starter-web: Mainly includes Spring MVC libraries and an embedded Tomcat server.
  * spring-boot-starter-test: Includes testing libraries like JUnit, Mockito, Hamcrest and Spring Test.
  * spring-boot-starter-data-mongodb: This dependency includes the MongoDB Java driver, and the Spring Data Mongo libraries.

&nbsp;

## <span style="color: #0b5394;">3 Application setup</span>

Thanks to Spring Boot, the application setup is as simple as the dependencies setup:

<pre class="lang:java decode:true ">@SpringBootApplication
public class AggregationApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(AggregationApplication.class, args);
    }
}</pre>

&nbsp;

When running the main method, we will start our web application listening to the 8080 port.

## <span style="color: #0b5394;">4 The repository</span>

Now that we have the application properly configured, we implement the repository. This isn’t difficult neither since Spring Data takes care of all the wiring.

<pre class="lang:java decode:true ">@Repository
public interface ProductRepository extends MongoRepository&lt;Product, String&gt; {
    
}</pre>

The following test proves that our application is correctly set up.

<pre class="lang:java decode:true ">@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = AggregationApplication.class)
@WebAppConfiguration
public class AggregationApplicationTests {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Before
    public void setUp() {
        productRepository.deleteAll();
    }
    
    @Test
    public void contextLoads() {
    }
    
    @Test
    public void findById() {
        Product product = new Product("LN1", "London", 5.0f);
        productRepository.save(product);
        
        Product foundProduct = productRepository.findOne("LN1");
        
        assertNotNull(foundProduct);
    }
}</pre>

&nbsp;

We didn’t implement save and findOne methods. They are already defined since our repository is extending MongoRepository.

## <span style="color: #0b5394;">5 The aggregation query</span>

Finally, we set up the application and explained all the steps. Now we can focus on the aggregation query.

Since our aggregation query is not a basic query, we need to implement a custom repository. The steps are:

#### <span style="color: #003366;">Implement the repository</span>

Create the custom repository with the method we need:

<pre class="lang:java decode:true ">public interface ProductRepositoryCustom {
    
    List&lt;WarehouseSummary&gt; aggregate(float minPrice, float maxPrice);
}</pre>

&nbsp;

Modify the first repository in order to also extend our custom repository:

<pre class="lang:java decode:true ">@Repository
public interface ProductRepository extends MongoRepository&lt;Product, String&gt;, ProductRepositoryCustom {
    
}</pre>

&nbsp;

Create an implementation to write the aggregation query:

<pre class="lang:java decode:true ">public class ProductRepositoryImpl implements ProductRepositoryCustom {
    
    private final MongoTemplate mongoTemplate;
    
    @Autowired
    public ProductRepositoryImpl(MongoTemplate mongoTemplate) {
        this.mongoTemplate = mongoTemplate;
    }
    
    @Override
    public List&lt;WarehouseSummary&gt; aggregate(float minPrice, float maxPrice) {
        ...
    }
}</pre>

&nbsp;

#### <span style="color: #003366;">Build the MongoDB pipeline</span>

Now we are going to implement the stages of the mongodb pipeline as explained in the beginning of the post.  
Our first operation is the <a href="https://docs.mongodb.org/manual/reference/operator/aggregation/match/" target="_blank" rel="noopener">match</a> operation. We will filter out all product documents that are beyond our price range:

<pre class="lang:java decode:true ">private MatchOperation getMatchOperation(float minPrice, float maxPrice) {
    Criteria priceCriteria = where("price").gt(minPrice).andOperator(where("price").lt(maxPrice));
    return match(priceCriteria);
}</pre>

&nbsp;

The next stage of the pipeline is the <a href="https://docs.mongodb.org/manual/reference/operator/aggregation/group/" target="_blank" rel="noopener">group</a> operation. In addition to grouping documents by warehouse, in this stage we are also doing the following calculations:

  * last: Returns the warehouse of the last document in the group.
  * addToSet: Collects all the unique product Ids of all the grouped documents, resulting in an array.
  * avg: Calculates the average of all prices in the group.
  * sum: Sums all prices in the group.

<pre class="lang:java decode:true ">private GroupOperation getGroupOperation() {
    return group("warehouse")
        .last("warehouse").as("warehouse")
        .addToSet("id").as("productIds")
        .avg("price").as("averagePrice")
        .sum("price").as("totalRevenue");
}</pre>

The last stage of the pipeline is the <a href="https://docs.mongodb.org/manual/reference/operator/aggregation/project/" target="_blank" rel="noopener">project</a> operation. Here we specify the resulting fields of the aggregation:

<pre class="lang:java decode:true ">private ProjectionOperation getProjectOperation() {
    return project("productIds", "averagePrice", "totalRevenue")
        .and("warehouse").previousOperation();
}</pre>

&nbsp;

The query is built as follows:

<pre class="lang:java decode:true ">public List&lt;WarehouseSummary&gt; aggregate(float minPrice, float maxPrice) {
    MatchOperation matchOperation = getMatchOperation(minPrice, maxPrice);
    GroupOperation groupOperation = getGroupOperation();
    ProjectionOperation projectionOperation = getProjectOperation();
    
    return mongoTemplate.aggregate(Aggregation.newAggregation(
        matchOperation,
        groupOperation,
        projectionOperation
    ), Product.class, WarehouseSummary.class).getMappedResults();
}</pre>

&nbsp;

In the aggregate method, we indicate the input class, which is our Product document. The next argument is the output class, which is a DTO to store the resulting aggregation:

<pre class="lang:java decode:true ">public class WarehouseSummary {
    private String warehouse;
    private List&lt;String&gt; productIds;
    private float averagePrice;
    private float totalRevenue;</pre>

&nbsp;

#### <span style="color: #003366;">Testing the application</span>

We should end the post with a test proving that results are what we expect:

<pre class="lang:java decode:true ">@Test
public void aggregateProducts() {
    saveProducts();
    
    List&lt;WarehouseSummary&gt; warehouseSummaries = productRepository.aggregate(5.0f, 70.0f);
    
    assertEquals(3, warehouseSummaries.size());
    WarehouseSummary liverpoolProducts = getLiverpoolProducts(warehouseSummaries);
    assertEquals(39.1, liverpoolProducts.getTotalRevenue(), 0.01);
    assertEquals(19.55, liverpoolProducts.getAveragePrice(), 0.01);
}

private void saveProducts() {
    productRepository.save(new Product("NW1", "Norwich", 3.0f));
    productRepository.save(new Product("LN1", "London", 25.0f));
    productRepository.save(new Product("LN2", "London", 35.0f));
    productRepository.save(new Product("LV1", "Liverpool", 15.2f));
    productRepository.save(new Product("MN1", "Manchester", 45.5f));
    productRepository.save(new Product("LV2", "Liverpool", 23.9f));
    productRepository.save(new Product("LN3", "London", 55.5f));
    productRepository.save(new Product("LD1", "Leeds", 87.0f));
}

private WarehouseSummary getLiverpoolProducts(List&lt;WarehouseSummary&gt; warehouseSummaries) {
    return warehouseSummaries.stream().filter(product -&gt; "Liverpool".equals(product.getWarehouse())).findAny().get();
}</pre>

&nbsp;

## <span style="color: #0b5394;">6 Conclusion</span>

Spring Data has a good integration with MongoDB aggregation framework. Adding Spring Boot to configure the application let&#8217;s us focus on building the query. For the building process, Aggregation class has several static methods that help us implement the different pipeline stages.

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.

&nbsp;