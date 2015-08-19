---
layout: post
title: Top 5 Redis use cases
author: joeengel
---

In this post, we'll explain some of the most common Redis use cases and different characteritics that are influencing these choices.

### 1. Session Cache

One of the most apparent use cases for Redis is using it as a session cache. The advantages of using Redis over other session stores, such as Memcached, is that Redis offers persistence. While maintaining a cache isn't typically mission critical with regards to consistency, most users wouldn't exactly enjoy if all their cart sessions went away, now would they?

Luckily, with the steam Redis has picked up over the years, it's pretty easy to find documentation on how to use Redis appropriately for session caching. Even the well-known ecommerce platform Magento has a [plug in](https://github.com/colinmollenhour/Cm_RedisSession) for Redis!

### 2. Full Page Cache (FPC)

Outside of your basic session tokens, Redis provides a very easy FPC platform to operate in. Going back to consistency, even across restarts of Redis instances, with disk persistence your users won't see a decrease in speed for their page loads—a drastic change from something like PHP native FPC.

To use Magento as an example again, Magento offers a plugin to utilize Redis as a [full page cache backend](https://github.com/colinmollenhour/Cm_Cache_Backend_Redis).

As well, for your WordPress users out there, Pantheon has an awesome plugin named [wp-redis](https://wordpress.org/plugins/wp-redis/) to help you achieve the fastest page loads you've ever seen!

### 3. Queues

Taking advantage of Redis' in memory storage engine to do list and set operations makes it an amazing platform to use for a message queue. Interacting with Redis as a queue should feel native to anyone used to using push/pop operations with lists in programming languages such as Python.

If you do a quick Google search on "Redis queues," you'll soon see that there are tons of open-source projects out there aimed at making Redis an awesome backend utility for all your queuing needs. Celery, as an example, has a backend using Redis as a broker that you can check out [here](http://celery.readthedocs.org/en/latest/getting-started/brokers/redis.html).

### 4. Leaderboards/Counting

Redis does an amazing job at increments and decrements since it's in-memory. Sets and sorted sets also make our lives easier when trying to do these kinds of operations, and Redis just so happens to offer both of these data structures. So to pull the top 10 users from a sorted set—we'll call it “user_scores”—one can simply run the following:

{% highlight bash %}
ZRANGE user_scores 0 10
{% endhighlight %}

Of course, this is assuming you're ranking users on an incremental score. If you wanted to return both the users and their score, you could run something such as:

{% highlight bash %}
ZRANGE user_scores 0 10 WITHSCORES
{% endhighlight %}

Agora Games has an amazing example, using Ruby, of a leaderboard using Redis as it's datastore that can be found [here](https://github.com/agoragames/leaderboard).

### 5. Pub/Sub

Last (but certainly not least) is Redis's Pub/Sub feature. The use cases for Pub/Sub are truly boundless. I've seen people use it for social network connections, for triggering scripts based on Pub/Sub events, and even a chat system built using Redis Pub/Sub! (No, really, [check it out](https://github.com/rajaraodv/redispubsub).)

Of all the features Redis provides, I feel like this one always gets the least amount of love, even though it has so much to offer users.

### Et tu?

So how about you, dear reader? Which use cases are *you* using Redis to solve?
