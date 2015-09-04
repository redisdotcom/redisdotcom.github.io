---
title: Backing Up Redis
author: billanderson
---

Backing up Redis is a complex question with both business and technical considerations. This guide will expose you to the basics of both aspects to help you determine if you need to back up your Redis, and if so what to consider when designing or choosing a solution.

### First Things First: Do You *Need* Backups?

For some people the question is absurd, yet it is one we often don't ask - and over-complicate our lives with unnecessary redundancy. I find the principle of implementing "as little as is possible, and no less" very useful here. Some scenarios have obvious answers. For example, are you caching web pages? No need for persistence, thus no need for backups. Redis is your primary data store on which your business is based? Of course you'll need to design and implement some form of backup for that. A good rule of thumb is "If you aren't persisting to disk, you don't need backups".

But the non-obvious cases, that is where it gets interesting. To determine if you need a copy of the data somewhere, consider the ramifications of losing the data? Is it easy to rebuild it? Then you probably don't need it. Is having the data live-replicated to other nodes suitable to continuity? If so, you probably don't want to bother with backups. Always ask the question, never assume you must have backup of the data. The rest of this guide assumes you have asked the question and the answer is "yes".

### Backup, Backup, Where Goes The Backup?

Now that you've established a need for them, where do you put them? What kind of backups do you need? Phase two of this process means getting a bit technical. You have on-premise backup, off-site backup, offline backup, hot backup, etc.. Which is appropriate for you? We'll take a quick tour through the "standard" options you have to explore what kinds of backup to choose from.

To aid in deciding what to use it may be helpful to cover some basic failure modes.

### Redis Server Failure Mode

While there exists a myriad of imaginable failure scenarios, they boil down to two primary failure types.

#### Process Failure

The first mode to consider is the server process dying, either through fault or shutdown. In this mode the host the daemon runs on is still up - just the process died. This can happen via clean shutdown, process panic, or even segmentation faults. While segfaults in Redis are rare, often due to bad ram or custom patching, they are possible.

#### Host Failure

In this scenario the host your Redis service is running on dies. Be it through kernel panic, DoS, tripping on a power or network cable, or even a truck running into the data center, the host is offline. Another possibility is the Linux OOM Killer killing Redis because the system is in dire need of memory space and Redis is using it, [although this can be addressed](http://backdrift.org/oom-killer-how-to-create-oom-exclusions-in-linux) and doesn't necessarily result in data loss / unavailability.

These two base cases form the root of all scenarios, so we will look at how to address these at the fundamental level.

### Option 1: Replication

Redis has a native replication capability. In this setup your data is replicated as it changes to one or more "slave servers" - servers which exist to be ready to become authoritative or be ready to provide the data for disaster recovery and can also be use for distributing read-only operations.  For more information see [Redis' replication documentatation](http://redis.io/topics/replication).

This option is one of the most simple among our choices. It is built into Redis and only requires a single additional process. It is also the quickest for recovery as you can point clients to the slave or, even better, change the backend node your load balancer sends traffic to. For managing this automatically, you can combine this setup with [Redis' Sentinel mode](http://redis.io/topics/sentinel).

The absolute minimal way to implement this is to have a second instance running on a different port, socket, or IP in the same node configured to replicate the master process' data. This would only protect against someone or something killing the master process, doing nothing for the host failure mode.

By placing the replicant on a different node one can provide protection against host failure.

### Option 2: Disk Persistence

While option one resides in wholly in memory, this option enables a copy of data being persisted to disk. This handles process failure differently and provides some level of host failure resiliency. With this option your data can reside on a single host, with a single instance, and survive a reboot. This single node setup can still suffer data loss through disk failure including loss of the host. This persistence can be enabled on your master server, all nodes, or only on (some or all) replicants - depending on your data and traffic load. For more information see [Redis' persistence documentatation](http://redis.io/topics/persistence).

### Option 3: Replication + Persistence

This is a simple combination of the prior options. It provides a means of having your data backed up to more than one host while storing it on disk as well. In this configuration you can survive the complete destruction of your main server. You have the data available on at least one other node which can reboot and still maintain the data.

### Option 4: All of The Above, Plus Out Of Band Backups

Now we get into highly paranoid levels of backup. In this scenario you spread your data across several replicants, likely running in different physical locations, with replicants persisting their data to a file on disk; then we take that file from each replicant and copy it to yet somewhere else. This last location could be some cloud storage provider, yet more systems in yet additional geographical locations, or even tape. At this point you are usually doing this for archival purposes rather than standard recovery (or general paranoia).

One aspect of this level of backups is managing multiple slave backups. If you have several slaves and are protecting each one, which one is the "correct" one to restore if you need to do a restore? There isn't the concept of a 'canonical' source.  It is not guaranteed that they will all be identical as the individual backups on each slave may not happen at precisely the same time. This brings some additional considerations when using this method, and you'll want to figure those out prior to the day you need to restore from this.

You may have noticed this list has increased in resiliency as it continues. It also happens to coincide with an increase in complexity and cost. When determining which of these options to choose remember to bias toward the minimum needed; one can add more if and when needs grow.

### OK, Which Option is Right For Me?

Here is where you need to look at your specific data and usage as it can limit the above options. In general, the larger your data the closer to the first portion of the list your options are limited to.

For example, if you have a large amount of data, out of band disk files will be less viable as there is time to save the data to disk which is then added to time to copy the data off of the system. This will lead you to the next bottleneck: how small your "Data Loss Window" (DLW) is.

Your loss window is the amount of time you can weather data loss. Since in the real world there is no such thing as instantaneous data replication (either to disk or another process), there is going to be some amount of time in which a failure means lost data. This is, incidentally, true for all systems not just Redis. Thus we have, in effect, a tug of war between "Thou Shalt Not Lose Data" and the Laws of Physics as we know them.

It is not uncommon for leadership or customers to react to the question of "How much data loss is acceptable" with "None!". However, given it isn't possible what we can do is reduce the Data Loss Window and contrast that with what it takes to save data: the "Data Persistence Window" (DPW).

For example if we say we can tolerate no more than one second of data loss, yet it takes more than 10 seconds to save the data. Now we have an issue - you simply can not save the data fast enough to meet the requirements. Imagine a slider. On the left end you have the DLW, and the right the DPW.

The larger the DPW, the larger the DLW will have to be. So you move the slider to the left. On the other hand,
the larger the DLW, the more room you have in persistence so when this is important you move the slider to the right. You can't move the slider to both ends at the same time, so you must find the balance between the ideals. This is where knowing your data size and availability requirements will come into play.

### How Often Do You Need Backups?

This is an important question because it can affect your solution. Usually the first answer which comes to mind is "all the time". Realistically this isn't viable. The above constraints prevent this.

How much data can you afford to lose in the case of a *complete* system failure? The emphasis here is on "can afford" rather than "in an ideal world". If you are using Redis for what it is designed for you are likely looking at daily backups, keeping perhaps a week or two of them.

This serves as a reasonable frequency to provide continuity in a disaster scenario. By combining this frequency with Redis' inbuilt replication capabilities, you can keep the risk of data loss low and provide some recovery for "oops" factors. That said, I can't emphasize enough how important it is to consider your actual requirements and filter them through what is physically possible, what is fiscally allowable, and what is technically acceptable.

When you apply these filters to your availble options, it is likely what choices to make will become obvious, or at least narrow them down to something simple enough to pass on to management for a business-level decision.
