+++
title = "How to build a virtualization solution?"
date = 2019-10-21T20:17:03+02:00
author = "Javi"
tags = ["libvirt", "kvm", "virtualization"]
showFullContent = false
draft = false
description = "How do virtualization solutions work? I’m curious about that, and I want to create one from scratch using only open source technologies. In this post, I will show you the main components of these types of solutions and their main responsibilities."
+++

This is a huge topic, but I will apply here the phrase *divide and conquer*. I will explain the main components of these solutions one by one, the topics around each one and the technologies that can be used. Remember that I'm not an expert on this topic, just a curious guy, and I can make mistakes. If so, leave me a comment please!, I will be happy to talk with you about this and exchange some opinions.

To make this post small and simple, I will define the scope of the virtualization solution that I will implement and deploy. I want to build a solution to manage my own home servers throught a web platform. Why not use one of the existing solutions in the market?. Well, I want to have fun and learn new things. To resume, I want a challenge!.

Note that this is the very first part of a series of blogs where I will put my steps designing and building this virtualization solution. Stay tuned!.

### Let's get started

If you ever used [Digital Ocean](https://m.do.co/c/6f2ea013e5c7), then you used the pretty face of a complete virtualization solution but on a big scale, also called IaaS (*Infrastructure as a Service*). The purpose of an IaaS is to abstract all hardware, servers, networks, storage etc and offer it as an online service. The topic is bigger, but we will maintain it small. Go go go!.

First, at a high level, we need to identify each component of the solution. Each of these components, produces logs, has security implications, needs to be available as much time as possible and has to respond to performance requirements.

For this solution, I will use an API as an entry point to manage virtual machines, storage, and networks. Who can invoke these backend operations?. Logically, just me :-). I was thinking of security using *Basic Auth* to handle authentication, but probably, I will use JWT to handle authentication and authorization to restrict the use of some methods of the API and be stateless. 

Who receives the payload from the API?. The host agent. This agent, installed inside the host, invokes methods from the *libvirt* library. Who can monitor the state of the virtual machines?. Only the guest agent, that runs inside the VM deployed from the host.

#### <u>Backend</u>

This is an application that runs multiple web services. The main purpose of this component is to manage hosts resources through the host agent. It is the main interface of the system. Also, this application is responsible for scheduling virtual machines in the current hosts using a scheduler algorithm that will check CPU, memory and energy consumption.

>Scheduler algorithm can be used with a policy file written (for example) in YAML to make it as more generic as possible to be able to implement multiple schedulers using the same codebase.

How the scheduler know where to put a VM?. Well, here exists multiple strategies to check host resource usage quota:

* The backend pull data from the host agent every `X` seconds.
* The host agent push the data directly to the backend every `X` seconds.

Which is the best strategy for this use case?. I think that here fit best the first option, pull fresh data from the agents of the cluster. The problems?. 

* If we have more than one backend to ensure HA, then the scheduler must be distributed.
* The backend needs to know the status URL to request from the agents. For this purpose, we need a service discovery software to register agents, increasing system complexity.

There are more problems, but these two are the most relevant for scheduler design. Note that the second point is not really a problem, because we need a 'host discovery' software to register new hosts automatically when available. Imagine that I buy a new server. I will execute a set of Ansible roles and scripts over the new host to install dependencies, update software, apply security rules, configure log rotation and... publish the new host to be available to the backend to schedule new resources into it!. I will talk about this topic soon.

The backend will make many more operations, and to handle all these things, it will use a database to maintain all configurations. Imagine that I need to list all snapshots from one VM, how can I do it without a database?. Probably the technology used for the database will be [PostgreSQL](https://www.postgresql.org/), but I don't discard other options. I need to research.

I don't know what tech stack I will use to develop this component, but I'm thinking in using [Kotlin](https://kotlinlang.org/) with [Micronaut](https://micronaut.io/) framework for building services.

#### <u>Host agent</u>

This software is a layer on top of libvirt library, that executes the commands sent by the backend. It covers all functionality to create resources like virtual machines, disks, and networks in the hosts that the agent is installed. Without this software, there is no possibility to create resources.

>The agent exposes a REST API that should be called **only** by the backend. The communication with the backend must be encrypted using TLS, and I will configure CORS to allow only calls from the backend. This API must be considered internal, and not for public usage.

Also, the agent is responsible (among other things) to handle all metrics sent by guests, and report it to the main reporting system to show graphics. This application should be developed in a programming language with high performance and minimum CPU overhead. One candidate is [Go](https://golang.org/).

This application exchanges data with other systems through different protocols, but mainly in *https*. For example:

* The backend sends a request the create a new virtual machine. The request from the backend ends in the host agent, which will schedule the VM in the best host that complies with the given policy.
* The host agent sends metrics from the guests every five seconds to the central reporting system. This action occurs without interaction with the backend, through scheduled tasks. This task should be able to be updated, and this update is triggered by the backend.

These are only two examples of data exchange with other components, but can be more. Normally these systems should be the backend, the guest agent and the central reporting system.

How this system should be running?. Probably, the best option here is to run this application as a service inside the host, using systemd.

Another good option is to expose this API through an Nginx server (configured as an inverse proxy) and delegate into it the rate limit control and other security and performance checks.

#### <u>Guest agent</u>

This application runs inside all virtual machines running in a host. Probably this software could be used with a systemd service (only in Linux guests), configured to be restarted automatically when an error occurs etc. The communication method between the host and the guest can be through *vsock*. 

There is one guest agent for [Qemu](https://wiki.libvirt.org/page/Qemu_guest_agent) that I need to research. To know more about the current features of the guest agent see [this](https://wiki.qemu.org/Features/GuestAgent).

The purpose of this agent is to report the internal state of the guest through different metrics: CPU, memory and network usage. Also, it will provide its current IP.

#### <u>Front, CLI and SDK</u>

In order to administer the solution, I will build a frontend in [Angular](https://angular.io/). Other interesting tools for this solution are to create a CLI tool (maybe in Go).