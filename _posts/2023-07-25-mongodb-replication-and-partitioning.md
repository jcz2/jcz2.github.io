---
layout: post
title: MongoDB replication and partitioning
date: 2023-07-25 17:00 +0100
---
<link rel="stylesheet" href="/assets/swiper-bundle.min.css">
<script src="{{ base.url | prepend: site.url }}/assets/swiper-bundle.min.js"></script>

In this post, I will provide a high-level overview of how MongoDB achieves high availability
and horizontal scaling through replication and partitioning.

Let’s say that in our free time, we like to cook and bake.
We want to share recipes with others so we decide to create a website where users can add and comment on their favorite recipes.

{:refdef: style="text-align: center"}
![image](/assets/mongodb-availability-and-scaling/app-architecture.png){:style="width: 65%; height: auto;"}
{: refdef}
The architecture is very simple.
We have a Mongo database where we store recipes and an application that fetches the recipes, renders the page, and returns it to the user.

{:refdef: style="text-align: center"}
![image](/assets/mongodb-availability-and-scaling/deployment-1.png){:style="width: 75%; height: auto;"}
{: refdef}
It's a hobby project and we don’t expect a lot of traffic. Without thinking too much about it we only deploy a single instance of Mongo database that will handle all the reads and writes.
Our website is online. We keep on adding recipes and we are seeing some traffic.
People even start contributing their own recipes.


Everything looks promising until one day the server with the database crashes.
The website goes down. Users can't access their favorite recipes.
The problem is that our current deployment lacks redundancy.
The moment our instance goes down, our website is offline.

{:refdef: style="text-align: center"}
![image](/assets/mongodb-availability-and-scaling/deployment-2.png){:style="width: 75%; height: auto;"}
{: refdef}

If we had another instance of the database, in case of a failure we could perform an automatic failover and redirect all the traffic to that instance, therefore increasing availability.
The way Mongo achieves redundancy is through a replica set.

### Replica Set

{:refdef: style="text-align: center"}
![image](/assets/mongodb-availability-and-scaling/replica-set.png){:style="width: 40%; height: auto;"}
{: refdef}

Replica set is a group of mongodb instances.
Its goal is to provide redundancy and automatic failover.
Replica set must have at least three members.
Only one member is the primary while all others are secondary members.
The primary accepts writes and usually it’s the only one serving reads as well.
Secondaries can serve reads but their main purpose is to keep up to date by replicating data from the primary and provide redundancy in case of failure.

### Failover
Let’s look at what happens in case of failure.
<div class="swiper" style="width: 75%">
  <div class="swiper-wrapper">
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/failover-1.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/failover-2.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/failover-3.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/failover-4.png"/>
    </div>
  </div>

  <div class="swiper-pagination"></div>

  <div class="swiper-button-prev"></div>
  <div class="swiper-button-next"></div>
</div>

Members of the replica set send heartbeats between each other to verify that other members are up.
Let's assume there is a network partition and the primary loses connection to other memebers. The heartbeat sent by a secondary will not receive a response within a specified period and a timeout will go off. The member which has its heartbeat timeout calls an election.

### Election
An election is a process used to elect a new primary.

- each election has a term id, identifying the election
- each (voting) member can cast only one vote in a specific election
- member with the majority of votes becomes the new primary
- in case of a tie, a new election begins

### Failover (continued)
Going back to the previous scenario, let’s see what happens.
<div class="swiper" style="width: 75%">
  <div class="swiper-wrapper">
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/failover-5.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/failover-6.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/failover-7.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/failover-8.png"/>
    </div>
  </div>

  <div class="swiper-pagination"></div>

  <div class="swiper-button-prev"></div>
  <div class="swiper-button-next"></div>
</div>

When the secondary calls an election, a new term begins with id 2.
The candidate votes for himself and then asks other members to vote for him as well.
Since it received the majority of votes, 2 out of 3, it becomes the new primary for term number 2.
When the old primary comes online, it will see that a new election happened with a higher term number.
It will demote itself to a secondary and start replicating changes from the new primary.
This is how failover happens.
Initially, writes were coming to the primary at the top.
After it failed, secondaries didn’t receive heartbeats in a timely fashion. They called an election and elected a new primary
which started accepting writes and serving reads.
Let’s now see an example where an election results in a tie.

### Election tie

<div class="swiper" style="width: 75%">
  <div class="swiper-wrapper">
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/election-tie-1.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/election-tie-2.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/election-tie-3.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/election-tie-4.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/election-tie-5.png"/>
    </div>
  </div>

  <div class="swiper-pagination"></div>

  <div class="swiper-button-prev"></div>
  <div class="swiper-button-next"></div>
</div>

We have a replica set with 5 members.
The primary becomes unavailable.
Because the secondaries did not receive heartbeats from the primary
the election timeout went off at two of them at the same time.
The election timeout is randomized to avoid situation like this but it can still happen.
A new term begins and candidates vote for themselves.
Then they send requests for votes to all other members.
In this scenario it happens that the orange member is closer to the other top one
so its request reaches it earlier than the request from the blue member.
Similar situation happens for the blue member and the bottom node.
The consequence is that the election results in a tie.
To win the election a candidate requires the majority of votes.
In this case 3 out of 5 possible votes, but at this point all votes have been cast.
What will happen is that since there is no primary
the election timeout will go off again and a new election will begin.
This process will repeat until the election succeeds and a primary is elected.

### Network partition

<div class="swiper" style="width: 60%">
  <div class="swiper-wrapper">
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/network-partition-1.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/network-partition-2.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/network-partition-3.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/network-partition-4.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/network-partition-5.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/network-partition-6.png"/>
    </div>
  </div>

  <div class="swiper-pagination"></div>

  <div class="swiper-button-prev"></div>
  <div class="swiper-button-next"></div>
</div>

Let’s look at another example.
What happens when a network partition occurs in a replica set of five members?
The partition prevents members on the left from communicating with members on the right.
The primary will realize it can’t reach the majority of the members. It will step down and become a secondary. The members on the other side of the network partition form a majority of the replica set therefore they will hold a new election and elect a new primary.

### Replica set fault tolerance

{:refdef: style="text-align: center"}
![image](/assets/mongodb-availability-and-scaling/replica-set-fault-tolerance.png){:style="width: 75%; height: auto;"}
{: refdef}

How many failures can a replica set handle?
To elect a primary majority of votes is required, so in case of 3 member replica set, we can lose one member. As you can see adding additional members does not always increase fault tolerance.

### Replica set - summary
- a group of Mongo instances consisting of one primary member and secondary members
- provides redundancy
- automatic failover
- election process is used to elect a new primary in case the previous one becomes unavailable 
- Mongo uses a modified version of the Raft consensus algorithm to implement election

## Horizontal scaling

<div class="swiper" style="width: 75%">
  <div class="swiper-wrapper">
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/scaling-1.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/scaling-2.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/scaling-3.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/scaling-4.png"/>
    </div>
  </div>

  <div class="swiper-pagination"></div>

  <div class="swiper-button-prev"></div>
  <div class="swiper-button-next"></div>
</div>

We get our website back online.
By learning from our mistakes, however, this time we deploy the Mongo database as a replica set to avoid downtime.
In case one of the members goes down, the traffic is automatically shifted to another member.
The replica set does it’s job. The availability of our website gets better.
The website is becoming popular and as a result more and more users are coming back to share and comment on their favorite recipes.
At some point the traffic is so high that it reaches the capacity
of the machines that our replica set is deployed on, resulting in poor performance.
We realize that to serve more traffic we need to increase the capacity of our system, we need to scale our database horizontally by adding more nodes.
To do this we need to partition (shard) our database.

### Sharding
Sharding is the process of splitting data into parts and distributing them across multiple shards.

{:refdef: style="text-align: center"}
![image](/assets/mongodb-availability-and-scaling/sharding.png){:style="width: 45%; height: auto;"}
{: refdef}

Initially each instance of mongo contained the whole dataset.
After sharding each instance contains only part of the data.
Whereas previously only one node handled all the traffic, now both reads and writes are spread across multiple nodes.
The question is how does mongo know how to split the data?

### Shard key
A shard key is a field or a group of fields used to distribute documents across shards.

```
{
 "title": "My grandma's original Carbonara recipe",
 "content": "In this recipe I would like to show you ...",
 "number_of_ingredients": 5,
 ...
}
```

Let’s say this is part of the document representing a recipe on our website.
It contains title, content, rating and other fields.
To shard the collection we can use the “number_of_ingredients” field.
Let's assume we only have recipes with a number of ingredients in range from 1 to 30.

<div class="swiper" style="width: 75%">
  <div class="swiper-wrapper">
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/shard-key-1.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/shard-key-2.png"/>
    </div>
  </div>

  <div class="swiper-pagination"></div>

  <div class="swiper-button-prev"></div>
  <div class="swiper-button-next"></div>
</div>

When we specify this field as the shard key, Mongo will take the span of possible values and split it into non-overlapping ranges. Each range of values is associated with a chunk.
A chunk contains all the documents whose shard key falls within the given range.
For example a recipe with 10 ingredients falls in the range of 10 to 15
and therefore will be contained in the 3rd chunk.
Mongo then distributes the chunks across shards.

{:refdef: style="text-align: center"}
![image](/assets/mongodb-availability-and-scaling/sharding-user-request.png){:style="width: 45%; height: auto;"}
{: refdef}

Now when users make reads and writes the requests will be directed to specific shards depending on the shard key, therefore spreading the traffic across multiple machines.
This sounds fairly straightforward but what happens if the majority of users are only interested in quick, cheap, and simple recipes with few ingredients?
That would mean that most of the traffic is being directed to shard 1 which contains the chunk with 1-5 ingredients recipes, whereas shards 2 and 3 are seeing barely any activity.
Let’s see what happens in that case.

### Chunk splitting

<div class="swiper" style="width: 75%">
  <div class="swiper-wrapper">
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/chunk-splitting-1.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/chunk-splitting-2.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/chunk-splitting-3.png"/>
    </div>
  </div>

  <div class="swiper-pagination"></div>

  <div class="swiper-button-prev"></div>
  <div class="swiper-button-next"></div>
</div>

Let’s say our users keep adding recipes with very few ingredients.
At some point, the chunk that holds this range will grow over the threshold which by default is 128 MB. That chunk will be split into two smaller chunks which cover the key range of the initial chunk.
If we keep on adding new documents at some point we will end up with ranges that contain only one value.
Since they contain only one value they cannot be divided any further.
Remember, a given range can only be contained by one chunk so we can’t split the range which only contains e.g. 4 ingredients into two chunks.
If we keep on adding even more documents to these chunks we will end up with what’s called a jumbo chunk.

### Jumbo chunk
A jumbo chunk is a chunk that has grown over the specified chunk size but cannot be divided
into smaller chunks.

{:refdef: style="text-align: center"}
![image](/assets/mongodb-availability-and-scaling/jumbo-chunk.png){:style="width: 30%; height: auto;"}
{: refdef}

Since we cannot split it into smaller parts we cannot distribute it over shards.
This means that all reads and writes for this specific shard key will be handled only by a single shard.
Let’s keep in mind the problem of jumbo chunks in the back of our heads and look for a moment at the properties of a shard key.

### Shard key properties
- cardinality - the number of distinct key values  
In our case, the cardinality is 30 since we have recipes with a number of ingredients that can go from 1 to 30.
- frequency - how often a given key value occurs  
In our example, the frequency of the values 5 to 10 will most likely be higher than other values since most recipes have between 5 and 10 ingredients.
- monotonicity - whether values of the shard key are increasing or decreasing


### Monotonicity
A key is monotonically increasing if each new value is higher or equal to the previous one.

<div class="swiper" style="width: 75%">
  <div class="swiper-wrapper">
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/monotonicity-1.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/monotonicity-2.png"/>
    </div>
  </div>

  <div class="swiper-pagination"></div>

  <div class="swiper-button-prev"></div>
  <div class="swiper-button-next"></div>
</div>

A key is monotonic if the value of each new key is either higher or equal to the previous one, or lower or equal.
For example, let’s say we used the time of adding a recipe as the shard key.
Each value will either be equal to the previous one if the recipes were added at the same time, or the next key will be higher.
Why is this a problem?
Because all of these documents will be added to one single chunk.
The one that contains the range with maxKey.
Therefore one shard will be responsible for handling all the new writes.

### Properties of a good shard key
- high cardinality  
We want a key to have high cardinality so that a range of values can always be split into smaller ranges if a chunk grows too much
- low frequency  
We want low frequency so that reads and writes are distributed uniformly across the cluster
- non-monotonic  
We want a key that is non-monotonic so that all the writes are not being handled by one shard

### Adjusting the shard key
```
(number_of_ingredients) -> (number_of_ingredients, created_at)
For example:
(5, 2023-02-02 10:00:35)
```

Looking back at our shard key, which is the number of ingredients in a recipe
we can see that it’s not a good key because it has low cardinality (it has only 30 possible values) and high frequency on the values between 5 and 10, although it’s not monotonic.
To make it a little bit better we can use a compound shard key that consists of the number of ingredients and creation date.
This will provide us with enough granularity to not run into the jumbo chunks issue and also split the traffic on the 5-10 ingredients recipes across different dates.
For example, all the 5 ingredient recipes that were added at a given time will have different
shard key than 5 ingredients recipes that were added at different times.
Let’s now have a quick look at what sharded cluster looks like.

### Components of a sharded cluster
- router - routes queries from clients to shards depending on shard key value
- shards - store chunks of sharded collections
- config servers - store cluster metadata, e.g. the location of each chunk

<div class="swiper" style="width: 75%">
  <div class="swiper-wrapper">
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/sharded-cluster-1.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/sharded-cluster-2.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/sharded-cluster-3.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/sharded-cluster-4.png"/>
    </div>
    <div class="swiper-slide">
      <img src="/assets/mongodb-availability-and-scaling/sharded-cluster-5.png"/>
    </div>
  </div>

  <div class="swiper-pagination"></div>

  <div class="swiper-button-prev"></div>
  <div class="swiper-button-next"></div>
</div>

The router uses the metadata from the config server to know which shard contains which chunk of a sharded collection.
Then based on the shard key in the request, sends the query to the proper shard.
In case the query doesn’t contain a shard key, the router sends the query to all shards
then gathers and combines the results before sending it back to the client.
To provide fault tolerance each shard is deployed as a replica set.

### Partitioning - summary
- allows horizontal scaling
- splits and distributes data across the cluster
- reads and writes are spread across many machines
- uses a shard key to partition the data
- uses a replica set for each shard to provide fault tolerance

Deploying Mongo as a sharded cluster helps us scale it horizontally and make it highly available.

### Bibliography
- [https://www.mongodb.com/docs/manual/](https://www.mongodb.com/docs/manual/)
- [https://www.oreilly.com/library/view/mongodb-the-definitive/9781491954454/](https://www.oreilly.com/library/view/mongodb-the-definitive/9781491954454/)

<script type="text/javascript">
  const swiper = new Swiper('.swiper', {
    speed: 0,

    pagination: {
      el: '.swiper-pagination',
    },

    navigation: {
      nextEl: '.swiper-button-next',
      prevEl: '.swiper-button-prev',
    },

  });
</script>