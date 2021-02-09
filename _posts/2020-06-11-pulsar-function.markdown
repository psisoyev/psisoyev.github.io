---
layout: post
title: Pulsar Functions in Scala
date: 2020-06-11 13:37:00 +0100
description: This post introduces Pulsar Functions in Scala
img: neutron-function/function.png
tags: [scala, apache pulsar, serverless, neutron]
comments: true
hidden: true
---

Data is important blablabla
2020 is rich for events

Today we are going to build a simple data analytics pipeline using [Apache Pulsar](https://pulsar.apache.org/) and [Neutron](https://github.com/cr-org/neutron).

Apache Pulsar is an open-source distributed pub-sub messaging system originally created at Yahoo and now part of the Apache Software Foundation. 
Often it is compared to Apache Kafka.
If you are interested in comparison of these two systems you can find several articles on this topic.

Also, we will use Neutron. It is a Pulsar client which is based on [FS2](https://fs2.io/) streaming library. 
Be aware, that Neutron is in active development and is not production ready.    

## The idea ??? better name
The whole idea is fairly simple.
A simple [application](???) will produce a stream of events and will write them to one of the Apache Pulsar topics.



Cheers!