---
title: The pitfalls of Redis
author:
  name: Ricardo Pieper
  link: https://github.com/ricardopieper
date: 2023-11-07 12:00:00 -0300
categories: [Redis]
tags: [redis, concurrent, distributed]
pin: true
---

Over many years, I've been using Redis for caching and as a "distributed data structure server". I love it. As an extremely lightweight in-memory database that Just Works™️, I've seen developers slap it onto a backend system as a caching system to make things go fast, reduce database load, or do distributed programming in some capacity.

I really like Redis beyond the usual caching scenarios. If your system is responsible for a funcionality that needs temporary data to operate in a distributed manner without persistence, it's great. You don't need a SQL database that could potentially be slow depending on what you're doing, or scheduling tasks onto a job queue to clean data after you're done. Redis has all of that, and also serves as a backend for job systems such as Quartz in Java or BullMQ in Node. Redis's documentation is also pretty good. All the commands are right in [this page](https://redis.io/commands/) and it's easy to locate the command you need.

Redis is amazing. But in this article I want to explore a bug I had where you need to be careful about how you store data in Redis and how you interact with it. This article requires some basic understanding of Redis to read. I will explore one of the bugs I saw in my carrer, but I had others that worked in a similar manner in other systems.

# Too much data in one key

When I started working at Monomyto Game Studios, one of the most annoying sporadic bugs was related to squad management. [Gunstars](https://gunstars.io/) is a battle royale top-down shooter game where you can join games in a squad or alone.  Sometimes, you would invite someone to a squad, the other person would accept, and only one of them would end up in the squad, while the other person would, IIRC, be left in an invalid state. They would join the squad, their session data would say as much, but the client wouldn't show them there.

In the backend, we had multiple servers, and each client could be connected in any of these servers. Inside the backend, the clients would communicate among themselves by sending messages. This was done using a framework called MagicOnion, but we don't need to go in depth in what MagicOnion does. Suffice to say, it made this communication easier. During the process of building a squad, both clients would add themselves to the same squad at the same time in Redis. The astute reader that does distributed programming on a daily basis (or even just concurrent programming on a shared memory architecture) can spot a potential problem right there.

The squad was represented in Redis as a single key, with a name similar to `squad_82506af6-2046-4688-a472-b31569ff974e` and contents like this:

```javascript
{
  "SquadLeader": "155afdaa-6b2d-4478-b10a-05094375f1bf" //the User ID of the squad leader,
  "GameMode": "DuoBattleRoyale",
  //...some other data,
  "Members": [
    {
      "UserId": "155afdaa-6b2d-4478-b10a-05094375f1bf",
      //...other data
    },
    {
      "UserId": "82506af6-2046-4688-a472-b31569ff974e",
      //...
    }
  ]
}
```

In the "other data", I think we copied some user profile information, like the current level of the character, their weapons, equipments, and so on. This was unecessary and some of that was removed. But it also would store some matchmaking information.

But other than that... this representation for a squad seems reasonable? 🤨 What's the problem?


## Concurrency in Redis

Redis is very fast, but runs single threaded. That fact seems to give developers free reign to just send commands to it without thinking about concurrent modifications to a key. Redis will just work, right?

However, the "single threaded", "sequential" part only really applies to modifying the state of the keys. You can't predict the order in which commands are sent in a distributed scenario without adding some sort of synchronization, which can be a huge can of worms. You can't predict exactly how fast the different concurrent tasks in your system will run, they might get preempted by the OS, they might just run a bit slower due to increased system load in one server while the other servers are running with lower load, and so on.

In Gunstars servers, the 2 connections, potentially in separate servers, would communicate among them about the squad invitation, and when the invitee accepted the invite, they would send a notification to the leader, and then add themselves to the squad. The leader, upon receiving that notification, would also add themselves to the squad. So we have a situation where both are adding themselves to the squad, in parallel.

This is how the invitee would behave:

1. Send the `InviteAccepted` event asynchronously, fire and forget

1. Without waiting for the leader to add themselves, we get the current squad data:
    ```redis
    GET squad_82506af6-2046-4688-a472-b31569ff974e
    ```

1. Deserialize, add themselves to the Members section (done in C#)

1. Serialize

1. Set the new squad data with `SET squad_82506af6-2046-4688-a472-b31569ff974e`:

    ```js
    {
      "SquadLeader": "155afdaa-6b2d-4478-b10a-05094375f1bf",
      "GameMode": "DuoBattleRoyale",
      "Members": [
        {
          "UserId": "155afdaa-6b2d-4478-b10a-05094375f1bf",
          //...
        }
      ]
    }
    ```

So far so good. How about the leader?

1. Receive the `InviteAccepted` event

1. `GET squad_82506af6-2046-4688-a472-b31569ff974e`

1. Deserialize, add themselves to the Members section (done in C#)

1. Set the new squad data with `SET squad_82506af6-2046-4688-a472-b31569ff974e`:

    ```js
    {
      "SquadLeader": "155afdaa-6b2d-4478-b10a-05094375f1bf",
      "GameMode": "DuoBattleRoyale",
      "Members": [
        {
          "UserId": "82506af6-2046-4688-a472-b31569ff974e",
          //...
        },
        {
          "UserId": "155afdaa-6b2d-4478-b10a-05094375f1bf",
          //...
        }
      ]
    }
    ```

Notice that in this example, it just worked. However, we got very lucky: the leader ran `GET squad_82506af6-2046-4688-a472-b31569ff974e` **after** the invitee added themselves. What if the invitee takes a bit longer to run, making both run `GET squad_82506af6-2046-4688-a472-b31569ff974e` exactly at the same time?


In that case, both would run `GET squad_82506af6-2046-4688-a472-b31569ff974e` and see the same state:
```js
{
  "SquadLeader": "155afdaa-6b2d-4478-b10a-05094375f1bf",
  "GameMode": "DuoBattleRoyale",
  "Members": []
}
```

Both would add themselves to that json:
```js
{
  "SquadLeader": "155afdaa-6b2d-4478-b10a-05094375f1bf",
  "GameMode": "DuoBattleRoyale",
  "Members": [
    {
      "UserId": "<either the leader or invitee>",
    }
  ]
}
```

And then both would store that key to Redis. Who will win that race?
```js
{
  "SquadLeader": "155afdaa-6b2d-4478-b10a-05094375f1bf",
  "GameMode": "DuoBattleRoyale",
  "Members": [
    {
      "UserId": "<Maybe the leader won? Maybe the invitee? Who knows!>",
    }
  ]
}
```
If you didn't understand the definition of a race condition, I hope that gives you an example beyond the basic multithreaded `x += 1` that is common to find in the internet.

Turns out in this case we are storing way too much stuff in one key. We can rework this and separate the members from the general squad data.

The solution
============

In a shared memory architecture, one could simply add a Mutex (or lock) on the squad data and be done with it. This makes all concurrent threads synchronize on the modifications to that list.

In a distributed memory architecture, we don't have that. Instead, what we need is a **distributed lock**. It just makes sense.

I am grug brained dev. I see multi thread race condition, I use lock. I see *distributed* race condition, I use *distributed* lock.

That would make it just work. There's even an algorithm called [Redlock](https://redis.io/docs/manual/patterns/distributed-locks/) that is implemented for many languages, including .NET. Let's just slap that shit on our codebase and we'll be done with it, right?

**No. Please don't.** That's not how [grugbrain.dev](https://grugbrain.dev/) works. grug brain do not like complexity. distributed lock very complex. me no like complex, me like simple. Let's think about what redis actually is: It's a *data structure server*. Data structures... sets, lists, hashes...

**Lists**. We can use a Redis List to solve that problem. Let's think about Redis being single threaded again: it means that operations on the Redis server state are applied sequentially. You can think of Redis's List commands like a buffed up version of `ConcurrentBag` in C# that also works in a distributed setting.

Let's get rid of the "Members" key in that JSON:

```js
 {
  "SquadLeader": "155afdaa-6b2d-4478-b10a-05094375f1bf",
  "GameMode": "DuoBattleRoyale",
  "MatchmakingStatus": "Waiting",
  //...
}
```

Now, when the `InviteAccepted` event is sent or received, each member can then simply add themselves to a new `squadmembers` list. The invitee would simply send this command:

```redis
LPUSH squadmembers_82506af6-2046-4688-a472-b31569ff974e 82506af6-2046-4688-a472-b31569ff974e
```

And the leader would send this command:

```redis
LPUSH squadmembers_82506af6-2046-4688-a472-b31569ff974e 155afdaa-6b2d-4478-b10a-05094375f1bf
```

The commands are interpreted by Redis sequentially, the `LPUSH` behaves correctly and does not overwrite the previous data. Now both members are in the squad.


## Atomicity

Rather than "too much data in one key", the main mistake here is thinking that, because Redis is single-threaded, operations will be magically atomic. But you have to remember: If a key can be modified by many parties at exactly the same time, you have to use atomic operations. However, I think that "too much data in one key" also means "too much *responsibility*" in one key, which is ripe for bugs.

That means you shouldn't read the whole data from Redis, modify it in the application code, and then write back.

```csharp
[App]   var newCounter = redis.Get<int>("COUNTER") + 1
[Redis] GET counter (Suppose it returns 0)
[App]   redis.Set("COUNTER", newCounter)
[Redis] SET counter 1
```

This works if there's only a single user modifying this counter, but suppose there's more:

```csharp
[User1] var newCounter = redis.Get<int>("COUNTER") + 1
[Redis] GET counter (Suppose it returns 0)
[User2] var newCounter = redis.Get<int>("COUNTER") + 1
[Redis] GET counter (returns 0)
[User1] redis.Set("COUNTER", newCounter)
[Redis] SET counter 1
[User2] redis.Set("COUNTER", newCounter)
[Redis] SET counter 1
```

This makes the counter 1 instead of what we really want, which is 2. The solution is to use an atomic operation, like INCR:
```csharp
[User1] var newCounter = redis.Incr("COUNTER")
[Redis] INCR COUNTER (returns 1)
[User2] var newCounter = redis.Incr("COUNTER")
[Redis] INCR COUNTER (returns 2)
```
You can't predict what a specific user's `newCounter` will be, but they will increase monotonically.

Network traffic
===============

Storing too much data in one key is also bad for performance. Redis can handle many thousands of operations per second, maybe millions. But what happens when you store too much stuff in one key?

One possible problem is too much network I/O, to the point of saturating the network capabilities of the server. When I worked on a chat system, we had a module with a "main loop" responsible for handing out chats to customer service agents, using an algorithm that relied on Redis heavily. Even when we had only a couple thousand users, I saw that system doing 150MB/s and still somehow working! I didn't work on that system directly but we all could see the amount of data in the monitoring systems.

Unfortunately, it's surprisingly hard for some newer developers to understand how much data there is in just 1MB of pure text. And no, that Redis instance was *not* streaming video, audio or transferring images. It was purely chat and some control messages for the handout of conversations for agents. Some estimates I found on the internet tell me 1MB is like a book with 500 pages. That's a lot! Imagine 150 of these books per second.

That system would not work for much longer if action wasn't be taken. It was also not a trivial problem to solve. A few months after we first saw it, COVID happened and rapidly the amount of users grew, so eventually it had to be solved or mitigated. I don't know what the solution was, but I'm fairly confident we had many keys such as these:

```js
{
  "status": "...",
  "name": "...",
  "email": "...",
  "channel": "whatsapp",
  //... many properties...
  "lastMessage": {
    //... a fairly big JSON object here
  }
}
```

This key would be fetched in its entirety even when only a status was necessary. Therefore many KBs worth a data would be fetched instead of just a few bytes, increasing the I/O to staggering levels for that amount of users. One way to quickly mitigate this problem is to create a read-only Redis replica and do reads from that server, this way the chat control system doesn't affect the other parts of the whole system. IIRC, eventually the keys had some redesign to remove rarely used properties from the main keys.


Conclusion
==========

Hopefully I demonstrated the concept of data races and atomicity in Redis in a way you can understand. Usually these concepts are only explained with shared memory architectures, with mutexes and so on. Turns out Redis does no magic and you will shoot yourself in the foot if you don't think about these concepts. I also hopefully explained the network traffic problem well enough so that you really think about what you're storing in Redis and how much of that you need on the hot paths of your system.

Here's a line adapted from Murphy's Law: If two users can modify a key at the exact same time, they eventually will, and the ensuing explosion will break your system in the worst way possible. That was a bit dramatic.
