---
layout: post
title: Using hashes in Redis
author: kylehunter
category: how-to
---

Hashes in Redis are a way to store associated field-value pairs under a single key, where both the field and values are strings. Redis allows for modifications to both the data structure as a whole, and also to each field in the structure. This makes it a great (and very fast) backing store for objects in an application.

### CLI Examples

Create a hash with two fields:

{% highlight bash %}
127.0.0.1:6379> HMSET my_hash key1 "foo" key2 "bar"
OK
{% endhighlight %}

List out the fields and values associated with the hash:

{% highlight bash %}
127.0.0.1:6379> HGETALL my_hash
1) "key1"
2) "foo"
3) "key2"
4) "bar"
{% endhighlight %}

Update the value of one of the fields:

{% highlight bash %}
127.0.0.1:6379> HSET my_hash key1 "xyzzy"
(integer) 0
127.0.0.1:6379> HGET my_hash key1
"xyzzy"
{% endhighlight %}

Delete a single field from the hash:

{% highlight bash %}
127.0.0.1:6379> HDEL my_hash key2
(integer) 1
127.0.0.1:6379> HGETALL my_hash
1) "key1"
2) "xyzzy"
{% endhighlight %}

Delete the hash:

{% highlight bash %}
127.0.0.1:6379> DEL my_hash
(integer) 1
127.0.0.1:6379> HGETALL my_hash
(empty list or set)
{% endhighlight %}

Redis's excellent documentation has more information about [the available commands](http://redis.io/commands#hash) for hashes. I won't copy/paste from there for you; instead, we're going to have a little fun with them.

### Dwemthy's Array!

[Dwemthy's Array](http://mislav.uniqpath.com/poignant-guide/dwemthy/) is a little example role playing game (RPG) written by Why the Lucky Stiff in [*Why's (poignant) guide to Ruby*](http://mislav.uniqpath.com/poignant-guide/) to help explain Ruby metaprogramming. Here, we're going to take the same RPG idea, and implement it using Redis hashes (with a little help from a Redis array).

We will create a hash for each monster in the array, looking something like this:

{% highlight yaml %}
name: "AssistantViceTentacleAndOmbudsman"
life: 320
strength: 6
charisma: 144
weapon: 50
{% endhighlight %}

Here is an example dive into the depths of the array:

{% highlight pycon %}
>>> import redis
>>> import dwemthy
>>> conn = redis.StrictRedis(host="localhost", port=6379)
>>> dw = dwemthy.Array.new(conn,
      {
        "name": "AssistantViceTentacleAndOmbudsman",
        "life": 320,
        "strength": 6,
        "charisma": 144,
        "weapon": 50
      }
    )
"[Get ready. AssistantViceTentacleAndOmbudsman has emerged!]"
>>> rabbit = dwemthy.Rabbit(dw)
>>> rabbit.sword()
"[You hit with 2 points of damage!]"
"[Your enemy hit with 10 points of damage!]"
"[Rabbit has died!]"
{% endhighlight %}

And here is the code as a whole, if you'd like to follow along: [dwemthy.py](https://gist.github.com/atarola/f0ae1df181d1611efcc9)

#### Code Highlights

Firstly, when a user creates the array with the factory method provided, we'll setup our data structures and return the new `dwemthy.Array` object:

{% highlight python %}
class Array(object):
    list_key = "dwemethys_array"

    ...

    @classmethod
    def new(cls, conn, *bad_guys):
        """ Create a new set of problems for our hero to boomerang! """

        conn.delete(cls.list_key)

        # Give each bad guy a cozy spot in the array!
        for bad_guy in bad_guys:
            key = uuid.uuid4()
            conn.hmset(key, bad_guy)
            conn.rpush(cls.list_key, key)

        dw_list = cls(conn)
        dw_list.next_enemy()
        return dw_list
{% endhighlight %}

The `conn.hmset` here creates a hash for the bad guy. Once that happens, we push the bad guy's key into a Redis list, which is being used here as a simple queue.

When all the bad guys are set up, we then make sure the one at the head of the array is ready with the ```Array.next_enemy()``` method:

{% highlight python %}
class Array(object):
    list_key = "dwemethys_array"

    ...

    def next_enemy(self):
        """ Bring out the next enemy for our rabbit to face! """

        # We're moving on!
        if self.current != None:
            self.conn.delete(self.current)

        enemy_key = self.conn.lpop(self.list_key)

        if enemy_key is None:
            # Like a boss!
            print "[Whoa.  You decimated Dwemthy's List!]"
        else:
            # Get the new enemy ready!
            self.current = enemy_key
            print "[Get ready. {} has emerged!]".format(self["name"])
{% endhighlight %}

The `self["name"]` on the last line is done by defining the `__setitem__` and `__getitem__` methods on the array object. This allows us to treat the array as mostly a python dict, and allows for easy fetching and modifying of the fields on the current bad guy:

{% highlight python %}
class Array(object):

    ...

    def __setitem__(self, key, value):
        self.conn.hset(self.current, key, value)

    def __getitem__(self, key):
        value = self.conn.hget(self.current, key)

        # hashes only store strings!
        if key != "name":
            return int(value)

        return value
{% endhighlight %}

### Conclusion

The myriad ways Redis allows us to modify hashes enables developers to use it to solve a large set of problems. Hopefully this post has given you ideas to use in your own projects.
