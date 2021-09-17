---
layout: post
title:  "01. Episode 1: Creating an Enterprise ready automation. Intro"
categories:  ['PowerShell', 'Azure', 'Automation']
tags:  ['PowerShell', 'Azure', 'Automation']
date:  2018-07-02 09:55:16 +03:00
beginps: '<div class="syn_wrapper"><pre class="brush: powershell;">'
endps: '</pre></div>'
---

When expanding  a Data center environment to Azure it is critical to build a toolset, which gives an ability to  perform day to day tasks with ease. Such a toolset should be able to integrate with existing CMDB and ITSM tools and processes. This allows existing teams manage new environment as they used to do it with existing one. In this series of videos and blog posts we will try to look closer at possibilities, Powershell provides, to achieve this goal.
I’m going to go talk about some tools, techniques and tricks I use when developing Powershell-based automation. We will also look at approach we can take to build enterprise-ready automation, so to speak.​

[![EPISODE 1: Intro](http://img.youtube.com/vi/7G8e27SMSZw/0.jpg)](http://www.youtube.com/watch?v=7G8e27SMSZw)

# Scenario

At the very beginning let’s define our scenario we are going to follow, during whole video and posts series. This scenario represents some general case, and we will use it as a context for our work.​

Our team works for a big company, and company just decided to expand to Azure. There are some expectations we’ve got from our management. First of all we don’t want to change our support organization significantly. So we want our existing teams to be able to support infrastructure in the cloud the same way the do it today.  We want to extend usage of our existing processes an tools to a new landscape. But at the same time we don’t want to loose flexibility of the Cloud. We want our developers to be productive, so we need to remove as much obstacles as possible, without affecting security.​
As part of this movement we were given a task to develop an automation toolset, which can be used to automate various support tasks for the cloud-based infrastructure. And as part of this video series we will try to look at this task from various angles.​

As part of the general task we were given the following Architecture guidelines. 

- The toolset should be cross-platform
- The toolset and infrastructure in the cloud should follow the Corporate Standards
- Infrastructure in the cloud should follow Azure best practices
- The toolset should provide as much flexibility as possible, with no harm to Corporate Security

First of all, our toolset should be cross-platform and reusable. The main idea behind it is that this toolset could be given to any team inside of the organization, so the team will be able to provision resources, manage their infrastructure and perform other automation tasks in a managed way. This means that the toolset itself and its counterparts in the cloud will make sure that everything that is done using this toolset follows the Corporate Standards. On the other hand the toolset and cloud-based infrastructure should follow the best practices provided for Azure. And of course we want to get as much flexibility as possible.​ But we dont want to lower our security level at the same time.

# Before you start

Before start implementing something in the cloud there are few things to do. Of course you need to plan network level, i mean connectivity, security and so on. On that field you need to find answers to the following questions:

- How cloud infrastructures are going to be connected (Express Route or VPN)?
- How to protect our internal networks from the one in the cloud?
- What topology you are going to use when establishing Data center WAN to the cloud connectivity?​
- How to control Internet access from your VMs in the cloud?
- What developer teams or other internal customers can do on networking layer in the cloud and what they are not allowed to do?

But this is not a the end, there are other questions :). These are just the most common ones.

However, we are not going to look at these questions in this article or in video series, at least for now. These are more theoretical and organizational questions, and i, perhaps, will cover them in one of later blog posts. Now we are going to focus on some practical things. And from practical automation on an Enterprise-grade lansdscape you first need to think of few things:

- Corporate Naming Standard
- CMDB and ITSM systems integration
- IPAM integration

## Corporate naming standard

Every component, or resource in Azure terms, has a name. Name represents your resource and, basically, is a part of unique identifier of a resource. This means that before you start developing your automation for corporate cloud environment, you need to extend your Corporate Naming Standard with cloud entities. In simple terms you need to develop a Naming Convention for cloud resources. For example, a naming convention may look like this:

```
AZ_REGION-BUCODE-DEPTCODE-RESOURCETYPE-SUFFIX
```

Having this kind of a name simplifies work for all people using such resources. They can easyly understand from a name, what this particular resource is. In addition to this, proper naming standard sets a context for automatic name generation procedures. And this allows building toolsets, which can generate names out of CMDB, without asking users to provide them, and, at the same time, making sure that names are unique. Of course there are limitations and requirements, which Azure sets for naming of various types of resources, some of them can be found [here](https://docs.microsoft.com/en-us/azure/architecture/best-practices/naming-conventions#naming-rules-and-restrictions). These requirements needs to be incorporated into the Naming Standard.
Once this Standard is set, we can integrate to CMDB to follow it in our automation.

## CMDB and ITSM systems Integration

As far as we wan to have our system manageable - we want everything to be in CMDB and connected to ITSM systems, including our ticketing systems, monitoring systems, event management systems etc. And on this path we need to make sure that out toolset is also integrated with CMDB. And the first step to build such integration, is to make it so, it can query and use CMDB data for name generation. CMDB contains whole org structure, so the toolset can use this information to generate names according to the Standard.

## IPAM Integration

When provisioning networks in the cloud, whichever topology we've chosen, we need to pick proper ranges from Corporate IP Namespace. For that our toolset should have an ability to talk to Corporate IPAM System and request ranges directly from it. This allow a lot of flexibility and speeds up provisioning greatly.

# Design of our DEMO toolset

In our case we decided to go with Powershell Core based toolset. Powershell Core is a cross-platform engine, which allows us to run scripts, and other more advanced Powershell stuff on each and every system you can think of. Very recently [AzureRM.Netcore module was released](https://channel9.msdn.com/Shows/Azure-Friday/Cross-Platform-for-Azure-PowerShell), and now it contains everything we need to build cross-platform toolset.

So at the very beginning we will look at what we can do with Powershell Core and AzureRM.Netcore. And for that we are going to implement CMDB integration and name generation based on it. But we want to set some goals for ourselves:

- Name generation should not depend on exact CMDB provider. We want to be able to easily replace one CMDB provider with another in our tool
- Name generation should be based on data from CMDB, and should take minimum number of parameters
- We want auto-complete parameters in our commands right from CMDB to avoid human errors

To achieve these goals we want to make our CMDB-aware components of the module - replaceable. So we can change our CMDB provider with minimal or no code change. Also, we need to be able to query our CMDB right from the powershell command line or from Powershell script or commandlet, to use thi data in name generation and in parameter auto-completion. For this we are going to develop a set of commandlets (cmdlets), which will do the trick.

# Future work
I'm going to put together the [video series](https://youtu.be/7G8e27SMSZw), i've already made some of them, which will describe and show the process live. All show various things i use when working with Powershell. But in addition to that i'm going to do a series of posts to describe these steps in more detail.