+++
title = "How to build a virtualization solution?"
date = 2019-10-21T20:17:03+02:00
author = "Javi"
tags = ["libvirt", "kvm", "virtualization"]
showFullContent = false
draft = true
description = "How do virtualization solutions work? I’m curious about that, and I want to create one from scratch using only open source technologies. In this post, I will show you the main components of these types of solutions and their main responsibilities."
+++

This is a huge topic, but I will apply here the phrase *divide and conquer*. I will explain the main components of these solutions one by one, the topics around each one and the technologies that can be used. Remember that I'm not an expert on this topic, just a curious person, and I can make mistakes. If so, leave me a comment please!, I will be happy to talk with you about this and exchange some opinions.

To make this post small and simple, I will define the scope of the virtualization solution that I will implement and deploy. I want to build a solution to manage my own home servers throught a web platform. Why not use one of the existing solutions in the market?. Well, I want to have fun and learn new things. To resume, I want a challenge!.

If you ever used [Digital Ocean](https://m.do.co/c/6f2ea013e5c7), then you used the pretty face of a complete virtualization solution but on a big scale, also called IaaS (*Infrastructure as a Service*). The purpose of an IaaS is to abstract all hardware, servers, networks, storage etc and offer it as an online service. The topic is bigger, but we will maintain it small. Go go go!.

### Let's get started

First, at a high level, we need to identify each component. Each of these components, produces logs, has security implications, needs to be available as much time as possible and has to respond to performance requirements.

For this system, I will use an API to manage virtual machines, storage, and networks. Who can invoke these backend operations?. Logically, just me :-). I was thinking of using *Basic Auth* to handle authentication, but finally, I will use JWT to handle authentication and authorization to restrict the use of some methods of the API and be stateless. Who receives the payload from the API?. The host agent. This agent, installed inside the host, invokes methods from the *libvirt* library. Who can monitor the state of the virtual machines?. Only the guest agent.

#### <u>Backend</u>

This is an application that runs multiple web services. The main purpose of this component is to manage host resources through the host agent. It is the main interface of the system. Also, this application is responsible for scheduling virtual machines in the current hosts using a scheduler algorithm that will check CPU and energy consumption.

>Scheduler algorithm can be used with a policy file written (for example) in YAML to make it as more generic as possible to be able to implement multiple schedulers using the same codebase.

#### <u>Host agent</u>

This software is a layer on top of libvirt library that executes the commands sent by the backend. It covers all functionality to create resources like virtual machines, disks, and networks in the hosts that the agent is installed.

#### <u>Guest agent</u>

This application runs inside all virtual machines running in a host.