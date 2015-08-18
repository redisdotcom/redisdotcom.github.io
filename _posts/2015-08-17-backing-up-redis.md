
---
Title: Backing Up Redis 
Subtitle: Why and when you should back up your Redis data 
Date: 2014-06-5
Tags: backup, persistence 
Author: Bill Anderson 
Summary: Backing up Redis is a complex question with both business and technical considerations. This guide will expose you to the basics of both aspects to help you determine if you need to back up your Redis, and if so what to consider when designing or choosing a solution.  
---

# First Things First: Do you *need* backups?

For some people the question is absurd, yet it is one we often don't ask
- and over-complicate our lives with unnecessary redundancy. I find the
principle of implementing "as little as is possible, and no less" very
useful here. Some scenarios have obvious answers. For example, are you
caching web pages? No need for persistence, thus no need for backups.
Redis is your primary data store on which your business is based?  Of
course you'll need to design and implement some form of backup for that.
A good rule of thumb is "If you aren't persisting to disk, you don't
need backups".

But the non-obvious cases, that is where it gets interesting. To
determine if you need a copy of the data somewhere, consider the
ramifications of losing the data? Is it easy to rebuild it? Then you
probably don't need it. Is having the data live-replicated to other
nodes suitable to continuity? If so, you probably don't want to bother
with backups. Always ask the question, never assume you
must have backup of the data.

The rest of this guide assumes you have asked the question and the
answer is "yes".

# Backup, Backup, Where Goes The Backup?

Now that you've established a need for them, where do you put them? What
kind of backups do you need? Phase two of this process means getting a
bit technical. You have on-premise backup, off-site backup, offline
backup, hot backup, etc.. Which is appropriate for you? We'll take a
quick tour through the "standard" options you have to explore what kinds
of backup to choose from.

To aid in deciding what to use it may be helpful to cover some basic
failure modes.

## Redis Server Failure Modes

While there exists a myriad of imaginable failure scenarios, they boil
down to two primary failure types.

### Process Failure

The first mode to consider is the server process dying, either through
fault or shutdown. In this mode the host the daemon runs on is still up
- just the process died. This can happen via clean shutdown, process
panic, or even segmentation faults. While these are rare, they are
possible.

### Host Failure

In this scenario the host your Redis service is running on dies. Be it
through kernel panic, DoS, tripping on a power or network cable, or even
a truck running into the data center, the host is offline.

These two base cases form the root of all scenarios, so we will look at
how to address these at the fundamental level.

## Option 1: Replication

Redis has a native replication capability. In this setup your data is
replicated as it changes to one or more "slave servers" - servers which
exist to be ready to become authoritative or be ready to provide the
data for disaster recovery.

This option is on of the most simple among our choices. It is built into
Redis and only requires a single additional process. It is also the
quickest for recovery as you can point clients to the slave or, even
better, change the backend node your load balancer sends traffic to.

The absolute minimum way to implement this is to have a second instance
running on a different port, socket, or IP in the same node configured
to replicate the master process' data. This would only protect against
someone or something killing the master process, doing nothing for the
host failure mode.

By placing the replicant on a different node you provide protection
against host failure. At the minimum level this is all still in memory
only.

## Option 2: Disk Persistence

While option one resides in memory, this option saves a copy of data to
the disk. This handles process failure differently, and provides some
level of host failure resiliency. With this option your data can reside
on a single host, single instance, and survive a reboot. This single
node setup can still suffer data loss through disk failure.

This persistence can be enabled on your master server, or only on
replicants - depending on your data and traffic load.

## Option 3: Replication + Persistence

This is a simple combination of the prior options. It provides a means
of having your data backed up to more than one host while storing it
on disk as well.

In this configuration you can survive the complete destruction of you
main server. You have the data available on at least one other node and
even that node can reboot and still maintain the data.

## Option 4: All of The Above, Plus Out Of Band Backups

Now we get into highly paranoid levels of backup. In this scenario you
spread your data across several replicants, likely running in different
physical locations, with replicants persisting their data to disk, and
finally we take that disk file from each replicant and copy it to yet
somewhere else. This last location could be some cloud storage provide,
yet more systems in yet additional geographical locations, or even tape.
At this point you are usually doing this for archival purposes rather
than standard recovery. 

You may have noticed this list has increased in resiliency as it went
along. It also happens to coincide with an increase in complexity and
cost. When determining which of these options to choose remember to bias
toward the minimum you need.

# OK, Which Option is Right For Me?

Here is where you need to look at your specific data and usage as it can
limit the above options. In general, the larger your data the closer to
the first portion of the list your options are limited to.

For example, if you have a large amount of data, out of band disk files
will be less viable as there is time to save the data to disk which is
then added to time to copy the data off of the system. This will lead
you to the next bottleneck: how small your "Data Loss Window" (DLW) is. 

Your loss window is the amount of time you can weather data loss. Since
in the real world there is no such thing as instantaneous data
replication (either to disk or another process), there is going to be
some amount of time in which a failure means lost data. This is,
incidentally, true for all systems not just Redis.

Thus we have, in effect, a tug of war between "Thou Shalt Not Lose Data"
and the Laws of Physics as we know them.

It is not uncommon for leadership or customers to react to the question
of "How much data loss is acceptable" with "None!". However, given it
isn't possible what we can do is reduce the Data Loss Window and
contrast that with what it takes to save data: the "Data Persistence
Window" (DPW).

For example if we say we can tolerate no more than one second of data
loss, yet it takes more than 10 seconds to save the data we now have an
issue. Imagine a slider. On one end you have the DLW, and the other the
DPW. 

The larger the DPW, the larger the DLW will have to be. The larger the DLW,
the more room you have in persistence.

You can't move them both in the same direction so you must find the
balance between the ideals. This is where knowing your data size and
availability requirements will come into play.

# How Often Do You Need Them?

This is an important question because it can affect your solution. Usually the
first thing which comes to mind is "all the time". Realistically this isn't
viable. The above constraints prevent this.

How much data can you afford to lose in the case of a *complete* system
failure? The emphasis here is on "can afford" rather than "in an ideal
world". If you are using Redis for what is is designed for you are
likely looking at once per day, storing perhaps a week or two of
backups.

This serves as a reasonable frequency to provide continuity in a
disaster scenario. By combining this frequency with Redis' inbuilt
replication capabilities you can keep the risk of data loss low and provide
some limit of "oops" factors. That said, I can't emphasize enough how important
it is to consider your actual requirements and filter them through what is
physically possible, what is fiscally allowable, and what is technically
acceptable.

When you run all of these filters into place it is very likely the answer will
be provided for you, or at least narrowed down to a few options which you can
provide to your management for a business level decision.
