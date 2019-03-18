---
title: "High Availability"
date: 2019-03-14T14:56:23+08:00
draft: false
---

One of the chief benefits of the Kubernetes open source container orchestration engine is how it brings greater reliability and stability to distributed applications, through the use of dynamic scheduling of containers. But how do you make sure that Kubernetes itself stays up and running, when a component, or even entire data center goes down?

People throw around “multi-master” and “high-availability” a lot. What’s your take?

There’s a difference between high availability and multi-master. If you, for example, have three masters and only one Nginx instance in front load balancing to those masters, you have a multi-master cluster but not a highly available one because your Nginx can go down at any time and well, there you go.
