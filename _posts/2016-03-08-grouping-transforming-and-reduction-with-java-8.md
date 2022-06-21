---
id: 11
title: Grouping, transforming and reduction with Java 8
date: 2016-03-08T07:38:00+01:00
author: xpadro
layout: post
permalink: /2016/03/grouping-transforming-and-reduction-with-java-8.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2016/03/grouping-transforming-and-reduction.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/2461519615137991925
categories:
  - Java
  - Java8
tags:
  - Java8
---

In <a href="http://xpadro.com/2016/02/multi-level-grouping-with-streams.html" target="_blank" rel="noopener">this previous post</a>, I wrote about how we can group collections of objects with streams and grouping. This is useful but does not cover specific use cases. For example, sometimes we do not only need to group things but also transform the result into a more appropriate object.

In this post, we will learn how to apply transformations and reduction to the groupingBy result.

<a href="https://github.com/xpadro/java8" target="_blank" rel="noopener">Here</a> you can view the source code of the following examples.

## <span style="color: #0b5394;">1 Grouping by and transform</span>

Let&#8217;s take the model I used in the previous post where we had a collection of <a href="https://github.com/xpadro/java8/blob/master/grouping/src/main/java/xpadro/java/grouping/model/Person.java" target="_blank" rel="noopener">persons</a> who owned a <a href="https://github.com/xpadro/java8/blob/master/grouping/src/main/java/xpadro/java/grouping/model/Pet.java" target="_blank" rel="noopener">pet</a>.

Now we want to know which pets belong to persons living in New York. We are asking for pets, so we can&#8217;t just make a grouping since we would be returning a collection of persons. What we need to do is group persons by city and then transform the stream to a collection of pets.

For this purpose, we use mapping on the result of the group by:

<pre class="lang:java decode:true ">public void groupAndTransform(List&lt;Person&gt; persons) {
    final Map&lt;String, List&lt;Pet&gt;&gt; petsGroupedByCity = persons.stream().collect(
        groupingBy(
            Person::getCity,
            mapping(Person::getPet, toList())
        )
    );
    
    System.out.println("Pets living in New York: " + petsGroupedByCity.get("NYC"));
}</pre>

&nbsp;

In the grouping phase, we group persons by city and then perform a mapping to get each person&#8217;s pet.

## <span style="color: #0b5394;">2 Grouping, transforming and reducing</span>

The previous example is useful for converting groupings of objects, but maybe we don’t want to obtain the whole list for each group. In this example, we still want to group pets by its owner’s city, but this time we only want to get the oldest pet of each list.

The method collectingAndThen from <a href="https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collectors.html" target="_blank" rel="noopener">Collectors</a> allow us to make a final transformation to the result of the grouping:

<pre class="lang:java decode:true ">public void groupTransformAndReduce(List&lt;Person&gt; persons) {
    final Map&lt;String, Pet&gt; olderPetOfEachCity = persons.stream().collect(
        groupingBy(
            Person::getCity,
            collectOlderPet()
        )
    );
    
    System.out.println("The older pet living in New York is: " + olderPetOfEachCity.get("NYC"));
}

private Collector&lt;Person, ?, Pet&gt; collectOlderPet() {
    return collectingAndThen(
        mapping(
            Person::getPet,
            Collectors.maxBy((pet1, pet2) -&gt; Integer.compare(pet1.getAge(), pet2.getAge()))
        ), Optional::get);
    }</pre>

&nbsp;

After we group persons by city, in collectingAndThen we are transforming each person in each city’s list to its pet, and then applying a reduction to get the pet with the highest age in the list.

## <span style="color: #0b5394;">3 Conclusion</span>

Java 8 grouping, transforming and reduction can be accomplished with the Collectors API not only allow us to group collections of things but also make transformations and reductions to obtain different objects depending on our needs.

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.