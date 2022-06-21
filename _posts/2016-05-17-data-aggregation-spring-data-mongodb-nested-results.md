---
id: 9
title: 'Data Aggregation Spring Data MongoDB: Nested results'
date: 2016-05-17T13:27:00+01:00
author: xpadro
layout: post
permalink: /2016/05/data-aggregation-spring-data-mongodb-nested-results.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2016/05/data-aggregation-spring-data-mongodb.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/3732573832160911402
image: /wp-content/uploads/2016/05/aggregationNotNestedResult.png
categories:
  - mongoDB
  - Spring
tags:
  - MongoDB
  - Spring
---

In a previous post, we built a basic example of an aggregation pipeline. Maybe you want to take a look at [Data aggregation with Spring Data MongoDB and Spring Boot](http://xpadro.com/2016/04/data-aggregation-with-spring-data.html) if you need more detail about how to create the project and configure the application. In this post, we will focus on learning a use case where it makes sense to group a portion of the result in a nested object.

Our test data is a collection of football players, with data about the league they belong to and how many goals they scored. The document would be like this:

<pre class="lang:java decode:true ">@Document
public class ScorerResults {
  
  @Id
  private final String player;
  private final String country;
  private final String league;
  private final int goals;
  
  public ScorerResults(String player, String country, String league, int goals) {
    this.player = player;
    this.country = country;
    this.league = league;
    this.goals = goals;
  }
  
  //Getters and setters
}</pre>

&nbsp;

It may be interesting to know how many goals were scored in each league. Also, who was the league&#8217;s top goalscorer. During the following section, we are going to implement our first simple example without using nested objects.

You can find the source code of all these examples at my <a href="https://github.com/xpadro/spring-data-mongo" target="_blank" rel="noopener">Github repository</a>.

## <span style="color: #0b5394;">1 Basic example</span>

We can use the following class to store each league&#8217;s result:

<pre class="lang:java decode:true ">public class ScorerNotNestedStats {
  private String league;
  private int totalGoals;
  private String topPlayer;
  private String topCountry;
  private int topGoals;
  
  //Getters and setters
}</pre>

&nbsp;

In order to retrieve the top scorers, we will first need to sort the documents by scored goals and then group them by league. In the repository, these two phases of the pipeline are implemented in the following methods:

<pre class="lang:java decode:true ">private SortOperation buildSortOpertation() {
  return sort(Sort.Direction.DESC, "goals");
}

private GroupOperation buildGroupOperation() {
  return group("league")
    .first("league").as("league")
    .sum("goals").as("totalGoals")
    .first("player").as("topPlayer")
    .first("goals").as("topGoals")
    .first("country").as("topCountry");
}</pre>

&nbsp;

That should do it. Let&#8217;s aggregate the results using Spring&#8217;s mongoTemplate:

<pre class="lang:java decode:true ">public List&lt;ScorerNotNestedStats&gt; aggregateNotNested() {
  SortOperation sortOperation = buildSortOpertation();
  GroupOperation groupOperation = buildGroupOperation();
  
  return mongoTemplate.aggregate(Aggregation.newAggregation(
    sortOperation,
    groupOperation
  ), ScorerResults.class, ScorerNotNestedStats.class).getMappedResults();
}</pre>

&nbsp;

If we retrieve the stats of the Spanish league, we get the following result:

<div style="clear: both; text-align: center;">
  <img loading="lazy" class="alignnone wp-image-243 size-full" src="http://xpadro.com/wp-content/uploads/2016/05/aggregationNotNestedResult_optimized.png" alt="Spring MongoDB Data aggregation - not nested results" width="332" height="116" srcset="https://xpadro.com/wp-content/uploads/2016/05/aggregationNotNestedResult_optimized.png 332w, https://xpadro.com/wp-content/uploads/2016/05/aggregationNotNestedResult_optimized-300x105.png 300w, https://xpadro.com/wp-content/uploads/2016/05/aggregationNotNestedResult_optimized-320x112.png 320w" sizes="(max-width: 332px) 100vw, 332px" />
</div>

<div style="clear: both; text-align: center;">
</div>

&nbsp;

Although this is fair enough, I don&#8217;t feel comfortable with all top scorer&#8217;s information scattered throughout the result class. I think it would make much more sense if we could encapsulate all scorer&#8217;s data into a nested object. Fortunately, we can do that directly during the aggregation.

## <span style="color: #0b5394;">2 Nesting the result</span>

Spring Data&#8217;s nested method is designed to create sub-documents during the projection phase. This will allow us to create the top goalscorer class as a property of the output result class:

<pre class="lang:java decode:true ">ProjectionOperation projectionOperation = project("totalGoals")
  .and("league").as("league")
  .and("topScorer").nested(
    bind("name", "topPlayer").and("goals", "topGoals").and("country", "topCountry")
  );</pre>

&nbsp;

In the line above, a nested document called topScorer is emitted by the nested method, which will contain all the data about the current league’s top goalscorer. Its properties are mapped to the output class using the bind method (topPlayer, topGoals and topCountry).

MongoTemplate’s invocation reuses our previous sort and group operations, and then adds the projection operation:

<pre class="lang:java decode:true ">return mongoTemplate.aggregate(Aggregation.newAggregation(
  sortOperation,
  groupOperation,
  projectionOperation
), ScorerResults.class, ScorerStats.class).getMappedResults();</pre>

&nbsp;

Executing this query will result in a much more compact result, having all top goalscorer’s related data wrapped in its own class:

<div style="clear: both; text-align: center;">
  <img loading="lazy" class="alignnone wp-image-237 size-full" src="http://xpadro.com/wp-content/uploads/2016/05/aggregationNestedResult-min.png" alt="Spring MongoDB Data aggregation - nested results" width="282" height="137" />
</div>

&nbsp;

## <span style="color: #0b5394;">3 Conclusion </span>

Spring Data MongoDB nested method is very useful for creating well structured output results from our aggregation queries. Doing this step during the aggregation helps us avoid having java code to post-process the result.

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.