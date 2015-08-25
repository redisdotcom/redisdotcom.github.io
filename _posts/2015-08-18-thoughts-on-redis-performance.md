---
layout: post
title: Thoughts on Redis Performance
author: billanderson
---

As I am afforded the privilege of speaking with many people and companies using Redis in a variety of use cases from simple caching to multi-terabyte sized setups the one topic I am asked to address more than any other is performance. Redis is different in how you approach performance. In many, if not most, database servers you try to improve performance. With Redis the goal is to not slow it down. This is a very different approach and requires a different mindset to take advantage of it.

### Performance Metrics - Is Latency King?

There are essentially two performance metrics you are primarily concerned with when using Redis: how many commands (or transactions) per second can I execute and how long do they take. When you break it down you find former is a secondary result from the latter. Since Redis is single-threaded how many ops/sec you can push is absolutely tied to how long they take. Thus, ultimately what matters is what I refer to as "command latency."

I consider one of Redis’ advantages, it’s simplicity. You issue commands, not queries. Commands are a far simpler route to pull data, and in this case the simplicity is reflected in the speed of the commands. It also allows developers to provide optimized commands as opposed to trying to optimize for various queries. This elegant simplicity is then afforded to programmers consuming Redis. This often means doing "query" type operations such as filtering a set, on the client side rather than having the server software do it.

While some feel this type of query is best done in a consistent way on the server, I am currently of the opinion this is no longer a preferred route. The reason is that bugbear we call "scalability". When you start running a "horizontally scalable" web service or site, for example, you’ll often find early on that this patter works fine. However, when you start handling "obscene" levels of traffic you quickly learn the database is a significant bottleneck. Shortly thereafter you learn this database, usually an SQL store such as MySQL, is not "horizontally scalable". You can’t simply add more.

### Command Latency

Of course, neither is Redis. However the difference here is that by keeping filtering logic, sorting, and anything you can’t execute in a single Redis command or at most a few commands, you aren’t loading the DB down with logic - and thus "things to process". Redis should be used as a "data store", not a traditional "database server". This is the first aspect of "don’t slow Redis down" you need to grok. It should not need to be stated that once you start down the path of Lua scripting, you run the risk of net lower performance. You might not see it in development when your traffic is low to moderate. However, when you hit large scales you’ll see it. And by then the logic is too often already baked in and becomes a heavy amount of technical debt to move into application code. We all know how much priority is often afforded to clearing technical debt.

This isn’t to say Lua scripting doesn’t have a place. It merely means it should be highly scrutinized. A good rule of the wrist on when a Lua script is or isn’t a net performance loss is to compare the cost of the additional round trips you’d have to execute if you handled that logic client-side. It is not, however, how long your client code can do it. Consider it this way: if you reduce your round trip cost by 2ms, but add 3ms in script execution time you went the wrong direction.

Here we need to be keenly aware of the nature of a single-threaded server. That 3ms script is blocking potentially hundreds (or even thousands) of commands while it executes. If you split those into a (pull) -> (logic) -> (pull) sequence the server is able to process additional requests during the "(logic)" phases. By keeping the logic in client code you preserve the concurrency inherent to multi-client based system. If you need transactions or the Lua scripting then of course use them. But don’t use them because they seem to make your code "easier". Always be aware of the concurrency performance hit, measure it, and make the choice consciously.

This leads us to the second rule of "Don’t Slow Redis Down": preserve concurrency by avoiding server logic via scripts. A side benefit is the ability to migrate to a Redis Cluster setup without rewriting or dumping your Lua scripts which operate on multiple keys.

### Other Ways Redis Can Be Slowed Down

There are a few system-level, or operational, aspects of running a Redis server which can slow Redis down. As with any server using I/O resource limits can slow Redis down. For example, if you need 8GB of network bandwidth, but have 1GB, it will be "slow" for you. If your daemon process is limited to fewer open sockets than you need concurrent connections time will be spent waiting for connections to be closed instead of executing commands.

Probably the two most common operational choices which cause Redis to slow down is to 1) put it on a virtual machine - especially a Xen hypervisor based one and 2) heavy disk persistence.

The first one is fairly well addressed in standard Redis literature: don’t put it on a Xen hypervisor VM. The second one, the persistence, appears to be but it doesn’t go far enough in my estimation.

There are, of course, the standard recommendations: use a local fast disk, ensure you have enough memory to handle the CoW dance when persisting - even running persistence on slaves only. These can indeed have effects ripple through to your command latency. What is missing is how to properly determine if they are.

One standard way to test this is to use the inbuilt latency testing. Before I get into the technical details of how to do that I want to address the larger "how" question. The main subset here is when to run these tests. Firstly, this test is likely to be meaningless if your data set is small. How small? Determine how long it takes to dump your data to disk. If we’re talking a second or two, maybe even ten, you’re less likely to get meaningful data. Of course, this is all predicated on you doing a BGSAVE rather than a SAVE.

### Persistence Latency

This leads us to the first principle of what I refer to as "persistence latency" - the time from command hitting the server to the result hitting some form of persistence: Find the least latency introducing persistence option you can. Depending on your network and disk either AoF or Slave server will be the lowest latency bumping persistence option, with RDB coming in last.

RDB will come in last primarily because it has an inbuilt delay of N changes over T seconds. So, assuming you have enough changes in that minimal duration (smallest default window is 60 seconds) your persistence delay is the interval T + the time to write the RDB to disk. If it takes 30 seconds to dump that memory to disk with the default 60 second interval your "persistence latency" would be (60+30) 90 seconds.

This does, however, beg the question of whether this persistence latency is a problem. There are two aspects to this question: 1) Is it sufficient for your business requirements? and 2) Does it slow down your Redis?

The first question is one I can not provide a generic answer which answers everyone. I can say, however, that if your requirements for persistence latency are tighter than the above formula reveals you’ll get the answers are likely to trend toward "use a slave and/or AoF" instead of RDB. This assumes you aren’t making obvious mistakes on the platform you run Redis on. Which brings us to the second aspect: does it slow down Redis?

For some the question of "how long does it take to save the RDB file" becomes paramount in their minds. In my view, this is a mistake. Does it matter how long it takes? The answer to this question is "does the save slow down Redis". If not, then you shouldn’t care if it takes 1 second or 1 hour. For example, consider Server A which takes 1000 seconds to save the RDB. While not saving the intrinsic command latency range is 30-100 microseconds. During the save this latency is still in the 30-100 microsecond range. In this case, it would be premature optimization to work on reducing this RDB save time under the notion you have poor Redis performance.

However Server B takes a mere 10 seconds, but the command latency jumps from 30-100us to 130-250us. Now you have a reason to be concerned with how long that RDB file takes to generate because the save is slowing Redis down and you want to minimize that. If that means faster disks, at least now you have reason to justify them - assuming that increase of around 100-150 microseconds causes performance anxiety for your application(s). If it doesn’t, then you’re back to premature optimization.

Now, as to how to measure that latency, I’ll go into more details in part two of this post as this one has become quite long already.
