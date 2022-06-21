---
id: 12
title: Multi level grouping with streams
date: 2016-02-29T07:10:00+01:00
author: xpadro
layout: post
permalink: /2016/02/multi-level-grouping-with-streams.html
blogger_blog:
  - www.xpadro.com
blogger_author:
  - Xavier Padro
blogger_permalink:
  - /2016/02/multi-level-grouping-with-streams.html
blogger_internal:
  - /feeds/6026073423954662509/posts/default/7183141736764405596
image: /wp-content/uploads/2016/02/singleGroup.png
categories:
  - Java
  - Java8
tags:
  - Java8
---

With Java 8 streams it is pretty easy to group collections of objects based on different criteria. In this post, we will see how we can make stream grouping, from simple single level groupings to more complex, involving several levels of groupings.

We will use two classes to represent the objects we want to group by: person and pet.

**Person.class** 

<pre class="lang:java decode:true ">public class Person {
    private final String name;
    private final String country;
    private final String city;
    private final Pet pet;
    
    public Person(String name, String country, String city, Pet pet) {
        this.name = name;
        this.country = country;
        this.city = city;
        this.pet = pet;
    }
    
    public String getName() {
        return name;
    }
    
    public String getCountry() {
        return country;
    }
    
    public String getCity() {
        return city;
    }
    
    public Pet getPet() {
        return pet;
    }
    
    @Override
    public String toString() {
        return "Person{" +
            "name='" + name + '\'' +
            ", country='" + country + '\'' +
            ", city='" + city + '\'' +
            '}';
    }
}</pre>

&nbsp;

**Pet.class** 

<pre class="lang:java decode:true ">public class Pet {
    private final String name;
    private final int age;
    
    public Pet(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() {
        return name;
    }
    
    public int getAge() {
        return age;
    }
    
    @Override
    public String toString() {
        return "Pet{" +
            "name='" + name + '\'' +
            ", age=" + age +
            '}';
    }
}</pre>

&nbsp;

In the main method we create the collection we will use in the following sections.

<pre class="lang:java decode:true ">public static void main(String[] args) {
    Person person1 = new Person("John", "USA", "NYC", new Pet("Max", 5));
    Person person2 = new Person("Steve", "UK", "London", new Pet("Lucy", 8));
    Person person3 = new Person("Anna", "USA", "NYC", new Pet("Buddy", 12));
    Person person4 = new Person("Mike", "USA", "Chicago", new Pet("Duke", 10));
    
    List&lt;Person&gt; persons = Arrays.asList(person1, person2, person3, person4);</pre>

&nbsp;

You can take a look at the source code at <a href="https://github.com/xpadro/java8" target="_blank" rel="noopener">my Github repository</a>.

## <span style="color: #0b5394;">1 Single level grouping</span>

The simplest form of grouping is the single level grouping. In this example we are going to group all persons in the collection by their country:

<pre class="lang:java decode:true ">public void singleLevelGrouping(List&lt;Person&gt; persons) {
    final Map&lt;String, List&lt;Person&gt;&gt; personsByCountry = persons.stream().collect(groupingBy(Person::getCountry));
    
    System.out.println("Persons in USA: " + personsByCountry.get("USA"));
}</pre>

&nbsp;

If we take a look into the map, we can see how each country contains a list of its citizens:

<div style="clear: both; text-align: center;">
  <img loading="lazy" class="alignnone wp-image-211 size-full" src="http://xpadro.com/wp-content/uploads/2016/02/singleGroup-1.png" alt="stream grouping - single level" width="469" height="218" srcset="https://xpadro.com/wp-content/uploads/2016/02/singleGroup-1.png 469w, https://xpadro.com/wp-content/uploads/2016/02/singleGroup-1-300x139.png 300w" sizes="(max-width: 469px) 100vw, 469px" />
</div>

&nbsp;

The result shows persons living in the specified country:

_<span style="font-size: x-small;">Persons in USA: [Person{name=&#8217;John&#8217;, country=&#8217;USA&#8217;, city=&#8217;New York&#8217;}, Person{name=&#8217;Anna&#8217;, country=&#8217;USA&#8217;, city=&#8217;New York&#8217;}, Person{name=&#8217;Mike&#8217;, country=&#8217;USA&#8217;, city=&#8217;Chicago&#8217;}] </span>_

## <span style="color: #0b5394;">2 Two level grouping</span>

In this example, we will group not only by country but also by city. To accomplish this, we need to implement a two level grouping. We will group persons by country and for each country, we will group its persons by the city where they live.

In order to allow multi level grouping, the groupingBy method in class <a href="https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collectors.html" target="_blank" rel="noopener">Collectors</a> supports an additional Collector as a second argument:

<pre class="lang:java decode:true ">public static &lt;T, K, A, D&gt;
    Collector&lt;T, ?, Map&lt;K, D&gt;&gt; groupingBy(Function&lt;? super T, ? extends K&gt; classifier,
                                          Collector&lt;? super T, A, D&gt; downstream)</pre>

Let’s use this method to implement our two level grouping:

<pre class="lang:java decode:true ">public void twoLevelGrouping(List&lt;Person&gt; persons) {
     final Map&lt;String, Map&lt;String, List&lt;Person&gt;&gt;&gt; personsByCountryAndCity = persons.stream().collect(
         groupingBy(Person::getCountry,
            groupingBy(Person::getCity)
        )
    );
    System.out.println("Persons living in London: " + personsByCountryAndCity.get("UK").get("London").size());
}</pre>

&nbsp;

If we debug the execution, we will see how people is distributed:

<div style="clear: both; text-align: center;">
  <p>
    <img loading="lazy" class="alignnone wp-image-212 size-full" src="http://xpadro.com/wp-content/uploads/2016/02/twoLevelGrouping-1.png" alt="stream grouping - two level" width="510" height="285" srcset="https://xpadro.com/wp-content/uploads/2016/02/twoLevelGrouping-1.png 510w, https://xpadro.com/wp-content/uploads/2016/02/twoLevelGrouping-1-300x168.png 300w" sizes="(max-width: 510px) 100vw, 510px" />
  </p>
</div>

## <span style="color: #0b5394;">3 Three level grouping</span>

In our final example, we will take a step further and group people by country, city and pet name. I have split it into two methods for readability:

<pre class="lang:java decode:true ">public void threeLevelGrouping(List&lt;Person&gt; persons) {
    final Map&lt;String, Map&lt;String, Map&lt;String, List&lt;Person&gt;&gt;&gt;&gt; personsByCountryCityAndPetName = persons.stream().collect(
            groupingBy(Person::getCountry,
                groupByCityAndPetName()
            )
    );
    System.out.println("Persons whose pet is named 'Max' and live in NY: " +
        personsByCountryCityAndPetName.get("USA").get("NYC").get("Max").size());
}

private Collector&lt;Person, ?, Map&lt;String, Map&lt;String, List&lt;Person&gt;&gt;&gt;&gt; groupByCityAndPetName() {
    return groupingBy(Person::getCity, groupingBy(p -&gt; p.getPet().getName()));
}</pre>

&nbsp;

Now we have three nested maps containing each list of persons:

<div style="clear: both; text-align: center;">
  <img loading="lazy" class="alignnone wp-image-213 size-full" src="http://xpadro.com/wp-content/uploads/2016/02/threeLevelGrouping-1.png" alt="stream grouping - three level" width="528" height="283" srcset="https://xpadro.com/wp-content/uploads/2016/02/threeLevelGrouping-1.png 528w, https://xpadro.com/wp-content/uploads/2016/02/threeLevelGrouping-1-300x161.png 300w" sizes="(max-width: 528px) 100vw, 528px" />
</div>

&nbsp;

## <span style="color: #0b5394;">4 Conclusion</span>

The Java 8 Collectors API provides us with an easy way to group our collections. By nesting collectors, we can add different layers of groups to implement multi level groupings.

I&#8217;m publishing my new posts on Google plus and Twitter. Follow me if you want to be updated with new content.

&nbsp;