---
title: Kafka message scheduler
date: 2021-09-03T17:21:00
hero: /post/2021/architecture/scheduler/images/hero.jpeg
excerpt: technical introduction to the kafka message scheduler
authors:
  - fkarakas
description: "Venez découvrir pourquoi et comment nous avons mis en oeuvre Kafka sur MYTF1"
---

kafka message scheduler

# Introduction

At MYTF1 we are in charge of publishing content to tf1.fr website. The contents are mainly videos, programs and articles of the brodacasted programs on tf1 tv channel.
These contents are managed in the backend with a CMS (content management system), and the videos are transcoded with a video workflow.
They MYTF1 website is a high traffic website, so in order to expose this data we are heavily relying on caching mecanism such as redis, elasticsearch, cloudfront, etc ...
Initially this data was "pushed" with batch programs which were in charge of exporting all the needed information available in the CMS database to all the cache repositories.
Today we are moving to a streaming events system with Apache Kafka, meaning that everytime an information is updated or created in the CMS, the data is processed and pushed immediately.
So we have many small programs (kafka consumers) which are triggered by new messages published in kafka topics. This works perfectly for real time events, but for scheduled events it was not straight forward
and we need to address this type of events scheduled for near or far future.

# Goal

What are these scheduled events ? For example a video on MYTF1 website can be available next week for a 2 months duration, a program can be online, so visible on the site, in couple of days, etc ...
The problem was that all our processing programs were triggered by a kafka message in a topic, so the challenge was : how to trigger this programs with a kafka message based on a scheduled date ? 
The challenge was to store somewhere the data and send to a specific topic the data (kafka message) in order to trigger the consumers in charge of processing this data.

# Design

Of course we could use some kind of scheduler framework/tool, not very optimized but it could work. But we didn't want to use any external database or system for the scheduling of these messages.
Apache Kafka is a store, so why not use a kafka topic to store this schedules ? :thumbsup
Next challenge, how to schedule this events ? we can simply write a program in go which can store all theses messages in memory and use a GO timer to produce the message in a topic. 
This can work but the memory footprint will be big, actually we only need to store in memory the events for the current day and scan the topic everyday to update the list of events to be scheduled for the current day.
This was the first draft design of the kafka message scheduler.

In summary the goal of the kafka message scheduler is to publish kafka messages to a specific topic with a specific id and payload in the future.
A schedule is just a kafka message with specific headers and a payload. The schedule id should be uniq and if you want to change the payload of your message, you just have to publish a new version of your message in the scheduler's topic. The topic can have many versions for your schedule, the scheduler will always take the latest one. 
There is a client lib in go for preparing a kafka message to be a scheduled message : https://github.com/etf1/kafka-message-scheduler/tree/main/clientlib

Basically, the scheduler reads the topic from the beginning at startup and every day at 00:00 am and it stores in memory the schedules planned for the current day. 
It will also watch new incoming messages and process only messages for the current day by adding, updating or deleting schedules in the memory store. 
Once a schedule is triggered, the schedule is deleted in the kafka topic with a tombstone message (nil payload).

So there is no other storage than the scheduler's kafka topic, no external db etc... All schedules are stored in a kafka topic.

# Technical implementation

## Scheduler message

First let's take a look at a schedule message stored in the scheduler topic.
```
Headers:
    scheduler-epoch: 1893456000
    scheduler-target-topic: online-videos
    scheduler-target-key: vid1
    customer-header: dummy
Timestamp: 1607918336
Key: vid1-online
Value: "video 1"
```

As you can see it is a regular kafka message but with some specific headers. These headers are:

- scheduler-epoch: the schedule meaning when the message should be produce in the target topic, expressed in EPOCH (number of seconds since Thursday 1 January 1970 00:00:00)
- scheduler-target-topic: the target topic to which the message should be produced at the scheduled date
- scheduler-target-key: the key for the produced message to the target topic, it is generally different of the scheduler message

They key of the schedule message should be uniq per event type and the payload is your payload to be produced.

Now let's take a look of the message produced for previously described schedule message:

```
Headers:
    scheduler-timestamp: 1607918336
    scheduler-key: vid1-online
    scheduler-topic: schedules
    customer-header: dummy
Key: vid1
Value: "video 1"
```

- scheduler-timestamp: is the timestamp of the scheduler message
- scheduler-key: the scheduler key
- scheduler-topic: the scheduler source topic

As you can see, the key has changed to match the target key specified in the original scheduler message but the payload was kept.

## Scheduler topic

As explained before the scheduler datastore is a kafka topic. In this topic each schedules should have a uniq key but multiple message with the same key can be present.
Why ? because if the payload of the schedule message changes in time, the new version should be produced in the topic. For deleting a schedule you simply need to produce
a tombstone message which is a message with a nil payload.

<figure 1: scheduler topic>

As you can see in the diagram the schedule S1 has 3 versions and the schedule S2 has been delete, so the scheduler will only trigger the schedule version V3 of the schedule S1.

## GO program

The kafka message scheduler is writtent in GO, and is using the official go client library from confluent (https://github.com/confluentinc/confluent-kafka-go). We decided to us the official
client because in the kafka system, the client has a big part of logic regarding failover and recovery, and even if the library is based on C, it was, from our point of view, a wiser choice.

<figure 1: scheduler internals>

As shown above, the scheduler has 5 main components:

### Missed event handler

It is actually in charge of reading the scheduler topic from the beginning and determine if there is any missed schedules. It is done by consuming the topic and 
storing in a map all found schedules, it will be then determine if there is any missed schedules. When a schedule is triggered a tombstone message is produced in the topic, so if this tombstone is missing
then it is a missed schedule. It cannot process the message as they are read, it has to buffer the messages because a delete schedule or a new version of the schedule can be present after the first read.
The second tricky part is to determine when to stop this failover recovery, because if there a continuous flow of new messages in the topic, the process will never stop. The idea here is to stop
when the message topic creation timestamp is lower than the current time minus 1 second.
This missed event handler is usefull if the scheduler is done for some time in the day or if the scheduler has been restarted.

### Live event handler

It is in charge of processing live schedules incoming in the topic. If it is for the current day they are processed, or if it impacts a schedule already planned for the current day.
It is bascially notifying for schedules that should be triggered in the current day.

### Timers

It stores all the schedule which are basically go timers `time.AfterFunc(...)`, it is a basic map and is in charge of creating/updating/deleting the map according to the notification
send by the missed and live event handler.

### Partition controller

Its role is to watch the partition assignment of the kafka consumers used in the scheduler and notify the Timers. This feature is available in the confluent client library.
`"go.application.rebalance.enable": true` if you enable this option the automatic assignment to partitions is disabled, you are simply notify that a new set of partitions is assigned to you.
You can do some processing and the call `Assign(partitions)` to effectively assign you go program.

Why do we need this ? imagine that you have a single instance of the scheduler and the scheduler topic has 3 partitions, all of these three partitions will be assigned to this single instance.
Let say that you want to popup a second instance, kafka will assign partitions to these 2 new instances maybe 2 partitions to the first one and 1 to the second one. Knowing that you will be able
to remove from the Timers store the schedules which are not handled by your instance.

### Ticker

The ticker will be in charge of notifying the timers to reset its database every night at midnight in order to process the schedules for the new day. It will also reset to 0 the offset of all scheduler consumers.
So a new Timers store will be populated with the schedules of the current day.

### REST API

Last but not least the REST API of the scheduler was introduced lately and exposes the current configuration of the scheduler and also all schedules currently planned ready to be triggered.
It is is used by the kafka message scheduler asmin which is an admin GUI for managing schedulers (https://github.com/etf1/kafka-message-scheduler-admin)

# Summary

Today we are using the scheduler on production for the site MYTF1, and as you can see it is relatively simple component but tricky at the same time "The evil is in the detail", so if you need more information
i invite to check the github repository (https://github.com/etf1/kafka-message-scheduler) for more detailed explanation. I hope this article was useful and interesting, and keep in touch for the article
on the kafka message scheduler admin.

Thank you
