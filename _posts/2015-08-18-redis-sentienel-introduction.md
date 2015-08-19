---
layout: post
title: Redis Sentinel Introduction
author: joeengel
category: 
---

Redis Sentinel provides a simple and automatic high availability (HA) solution for Redis. If you’re familiar with how MongoDB elections work, this isn’t too far off. To start, you have a given master replicating to N number of slaves. From there, you have Sentinel daemons running, be it on your application servers or on the servers Redis is running on. These keep track of the master’s health.
