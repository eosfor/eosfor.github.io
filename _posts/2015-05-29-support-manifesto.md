---
layout: post
title: Support Manifesto
date: 2015-05-29 08:25:54.000000000 +03:00
type: post
categories: [SupportManifesto, PowerShell]
tags: [SupportManifesto, PowerShell]
---

Nowadays there is a big gap between developers and infrastructure operations. The problem is that they exist in two different worlds. They use different terminology and approaches, have distinctive strategies and goals, varying sets of knowledge, and even a different mindset. How can we work together like this, and who can join us?

We should all think not only in terms of our technical field, but also about the people we are doing this for. Customers don't just require a solution itself. They also need to support and monitor this solution, manage it, integrate it with existing solutions and data flows, deploy new instances and restore failed ones. When a solution contains a number of components that can run on various servers and be moved from one to another, there is a lifesaving property that should be provided – the system should be self-descriptive. In other words, it should provide some interface to track its state and components in an easy way.

How can we achieve this? Here are some ideas and technologies to use for developers who want to do something really great.

<!--more-->

## Manageability Strategy – give us tools and docs!
![img]({{ site.baseurl }}/images/posts/oldposts/052915_0825_supportmani1.png)

When a solution runs a lot of instances across a number of boxes, it is crucial to have some kind of  *scriptable* command line interface to manage it. This command line based toolset should provide management and enumeration functionality with which it should be easy to perform tasks like the ones below:

- Return all the components of the solution running on the server, by provided name of the server. Simple pseudo code example might look like this:*Get-AppComponents –Computername my.server.com*. This gives the ability to automatically build a map of the running components and their statuses.
- Perform basic operations on these components, like stop, start, restart, retrieve status and current configuration options, and reconfigure the component remotely.

All of this can be accomplished via the WMI interface provided by the solution and its components. It is possible to generate PowerShell cmdlets based on WMI if we have them. With IIS, there isn't even the need for a provider, because IIS offers enough tools for all of these tasks. Only some wrappers over the existing commands should be provided as part of the solution. Besides that, having WMI will give us the option to automatically discover the components of the application in a simple way.

## Hey Ops guys, here is the documentation you asked for
![img]({{ site.baseurl }}/images/posts/oldposts/052915_0825_supportmani2.jpg)

Thus, in order to make the solution manageable, developers should:

- Provide a command line based, scriptable management interface for the whole solution. On Windows, this should be accomplished via providing PowerShell based interface OR providing WMI interface. It can be easily converted to cmdlet using CDXML
- Provide technical documentation describing the high level overview, map and workflow between the components, the protocols used to communicate, and details on the internals of the components

## Monitorability Strategy
![img]({{ site.baseurl }}/images/posts/oldposts/052915_0825_supportmani3.png)

To be able to maintain good quality of service provided by the solution, it should be easy to monitor and react to incoming incidents. Besides, it should easily integrate into the existing monitoring solution of the enterprise. So how can we achieve this? There are three main ways to do it, from simple to complicated, and they are: event logs, performance counters, and event tracing for Windows.

Event log is the easiest way to provide data to all the existing monitoring systems. If the application puts a message in the log, it could be taken by a monitoring system and processed accordingly.

On the other hand, sometimes support, as well as development, requires additional information about the performance of the solution. For that, the solution should provide performance counters for its key points. But there is more, monitoring systems can also consume this data and generate alerts based on them. This gives the ability to quickly find the problem and react to it. Moreover, it becomes possible to gather statistics and look for ways to improve the solution. As a result, you have monitoring out of the box with no effort, no 3rd party tools, and no additional costs.

But, at some point, even this is not enough. You might need to look deeper into the solution and find how much time the solution spends inside of function calls. There are some situations where there is a need to correlate the events happening in the system at a particular point or during a particular period of time to find out the cause of slowness. And that is what Event Tracing for Windows (ETW) was introduced to do. If the solution provides information of this depth, it becomes possible to do deep dives into performance issues using nice looking graphical tools, and making different slices on different angles of view.

In summary, to make a solution fit into [ITIL](https://en.wikipedia.org/wiki/ITIL) and the existing monitoring, incident management, and problem management, and to change management policies, developers should:

- Consider using one of the standard ways to register events in the system.
- Consider providing key performance metrics via system performance counters.
- Optionally, make [Event Tracing for Windows](https://msdn.microsoft.com/en-us/library/windows/desktop/bb968803%28v=vs.85%29.aspx) providers for the product.

## Deployment Strategy
![img]({{ site.baseurl }}/images/posts/oldposts/052915_0825_supportmani4.png)

Here is the thing, deployment should be planned at the very beginning, at the point when the application architecture is just being built. This is the main prerequisite for easy deployment. There is another cornerstone of easy deployment on the Windows platform – packet based deployment. Having packets gives the option to rollback failed deployment out of the box. There are other benefits, like logging, temporary admin rights, uninstallation, centralized management with enterprise level, self-repair, setup customizations, the ability to advertise packages, and simple deployment based on GPO. So when the deployment method for the solution is the MSI packet, all of the above is provided out of the box.

These two technologies are on the cutting edge of deployment: Desired State Configuration (DSC) and OneGet.
- DSC technology is a way of making complex deployment workflows in a declarative way. The idea is to define the configuration of the server in a simple declarative way and request the server to configure itself. The main scenario is when servers query the main configuration point for newer configurations. If there are some, servers download them along with the necessary files and apply them. The benefit here is that this functionality is a part of the OS and is provided by the OS vendor. You don't need to use a 3rd party. The only thing that is required is a piece of logic called DSC Resource, which knows how to deploy this particular component. Make it, the DSC Resource code, put it on the configuration server and voilà – all your components and apps get configured automatically.
- [OneGet](https://github.com/OneGet/oneget) is a cutting edge experimental approach based on PowerShell. Think of it as a packet management system for Windows. In general, it is a framework that can be attached to any repository; by default, it is the Chocolatey public repository based on NuGet through a PowerShell-based or C#-based repository provider. If you have a provider framework, you can query the repository for packages and manage them.

At this stage, the workflow becomes simple and clear. The developer team should simply build the package and put it into the package repository. All the rest is automated by DSC and OneGet.

In conclusion, I should point out that for deployment to the existing enterprise infrastructure, developers should:
- Check if there is a source control/repository/deployment infrastructure in place and consider it as a default deployment platform
- Architect their solutions with that in mind

When there is no such infrastructure in place, the points to remember are:
- Use the platform provided package mechanisms as a default way of deployment.
- Make a central repository for packages.
- Make a package management solution based on OneGet or PowerShell.
