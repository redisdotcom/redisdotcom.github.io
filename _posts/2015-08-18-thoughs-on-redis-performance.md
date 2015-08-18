---
layout: post
title: Thoughts on Redis Performance
author: billanderson
category: 
---

As I am afforded the privilege of speaking with many people and companies using Redis in a variety of use cases from simple caching to multi-terabyte sized setups the one topic I am asked to address more than any other is performance. Redis is different in how you approach performance. In many, if not most, database servers you try to improve performance. With Redis the goal is to not slow it down. This is a very different approach and requires a different mindset to take advantage of it.
