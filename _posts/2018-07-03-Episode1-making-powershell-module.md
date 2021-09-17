---
layout: post
title:  '02. Episode 1: Making PowerShell module'
categories:  ['PowerShell', 'Azure']
tags:  ['PowerShell', 'Azure']
date:  2018-07-03 11:17:59 +03:00
beginps: '<div class="syn_wrapper"><pre class="brush: PowerShell;">'
endps: '</pre></div>'
---

Hello colleagues, in this post I'm going to talk about PowerShell modules. However, I don't want to write usual things everyone knows about. There are a lot of information about them in the [help files](https://docs.microsoft.com/en-us/PowerShell/module/microsoft.PowerShell.core/about/about_modules?view=PowerShell-6) and [on the Internet](https://kevinmarquette.github.io/2017-05-27-PowerShell-module-building-basics/). What I'd like to talk about is why you may want to use them, and what I learned from my experience about them.

<!--more-->

## Two common patterns I use

Most commonly modules fall into two patterns or use cases. First is, what I call "application management" or "local" pattern. This happens when you want to manage an entity or even multiple entities running on some servers or VMs. Typically you may want to deploy then, check their status, apply some changes to their configs and so on. Or it could be a set of standard actions you want your support teams to be able to perform across server estate. Or both :).

Second case is, what I call "remote case". In this scenario you, by some reason, are not able to run any code, locally on the system, where your entity executes. Example is - cloud. You can only manage it by using APIs given by cloud provider. In general, you may have some remote system you use for your needs and some management APIs, provided by that system.

## "Local" pattern

This pattern is very powerful option. Frankly speaking at the very beginning, when I just started making some automations, a dozen years back, I was sure that every single action on remote system should be performed remotely. I went through using various tools and technologies for this. I used Visual Basic, Windows Scripting Host, PSExec, WMI ans some Native API. I was making my own command line tools for that. Later, with PowerShell V2, I totally switched to it. But for me it always looked like it was set in stone - every single thing can be done remotely and I don't need any local tools on a remote system. But through the years of experience I changed my mind. And there are few reasons behind it. First of all is complexity. Sometimes it is much easier to write a piece of code which does something locally on the system, then do the same stuff remotely. It is not only measured by number of steps you need to do, or number of lines you need to write. It as also measured by logical complexity, especially when it comes to writing code, which handles failures of the process on remote side, or network fluctuations and failures. Sometimes you need to develop your own orchestration mini-framework, for handling all of this. In most cases it worth no time spent on it. At some point I realized, that when you need to do something really complex, logically heavy on some remote system, it is much easier and faster to develop something that does this thing locally, and then just start this piece of logic, initiate it from a remote system. I tried this approach on complex environments having hundreds of systems running across continents an it worked. It is is easy to develop and easy to debug. Go try debugging code running on remote system, returning values to your orchestrator and then sending new commands, base in received data. It is, well, it is not hard, it is slow. It eats time. When something breaks or simply gets stuck, you start over and over again. And at some point I just decided - this should not continue any longer
So now, in case I need to manage some component running remotely I write a code, which assumes it sits right next to the object it wants to manage. Right on the same host. And I develop out of this assumption. And PowerShell module is a perfect thing to put your code in, for such a case. Few reasons for that:

- It can be delivered from a central module repository as a package natively, by PowerShell itself, out of the box.
- It is boxed. You do not care about place to store it to, or any environment variables like PATH or something
- It is simple to write and debug
- You can embed help into it

And when I need to do something to this remote component running somewhere on remote system I only need to connect to that system and start a commandlet I need. All logic runs locally, so it is not affected by network, stars or position of Moon, it can even survive reboots if needed. So the main usage pattern for me now is WinRM + local PowerShell module. I should also mention here that very recently [PowerShell has gotten an ability to perform remote executions via SSH](https://docs.microsoft.com/en-us/PowerShell/scripting/core-PowerShell/ssh-remoting-in-PowerShell-core?view=PowerShell-6), so now you can follow this pattern even with Linux systems.

Once again, to sum everything up. When you have some component you want to manage running somewhere on remote systems, go with WinRM(SSH) + local PowerShell module. This is the simplest option you need.

## "Remote" pattern

This pattern makes sense only when you have only an API, provided by remote system. Like Azure REST API, or SQL/TSQL, or other similar stuff. Do not do this with files, processes, services, IIS and other similar things. Of course they provide APIs, but they also can be managed locally! So, in case you have such APIs, what are the benefits of using modules? Well, they are almost the same as with "local", but there are few things to point out. First of all is "embedded help". You should write extensive help for your cmdlets. You can either directly embed your help int your commandlets, using [comment-based help](https://docs.microsoft.com/en-us/PowerShell/module/microsoft.PowerShell.core/about/about_comment_based_help?view=PowerShell-6). Or you can use [platyps](https://github.com/PowerShell/platyPS) for this. You can find more about it in [this presentation](https://youtu.be/zGOl5g_AJ5U). Providing help is crucial. Especially for teams or individuals, who only start using PowerShell or your module. They simply afraid that they do something wrong. So, if you want your module to be used by your tem - add help to it.

Another really useful and, sometimes, even critical thing - parameter value auto-completion. If it is possible - provide it. We will look at how to do this in one of upcoming videos. For example imagine you had some commandlet you used to create subnets in a VNET under some Corporate Azure Subscription. First of all you needed to provide a Subscription name to it, then you needed to find a VNET and then give subnet a name. If you going to go through the process of finding all related information manually via portal or script - it will be time consuming. Imagine you needed to configure tens on subnets, for example. And the biggest problem is it will be subject for human error. But imagine how much easier for a user it could be if your commandlet would autocomplete a Subscription name, then, based on a Subscription it would show all VNETs under it, and then it would generate a name for you? This will eliminate even a possibility of making an error. This is all possible to do, especially if you have this information in CMDB and it is cached in the module.

So as a bottom line to this section, for "remote" modules destined to use by humans, provide detailed help and use auto-completion extensively. You need to integrate your module with CMDB to achieve good results.

## Module creation process

Well, this part is simple, and I usually use [plaster](https://github.com/PowerShell/Plaster) for this. It basically generates baseline for a module. I described this here:

[![EPISODE 1: Empty Module](http://img.youtube.com/vi/Wpy0oad_JOk/0.jpg)](http://www.youtube.com/watch?v=Wpy0oad_JOk)
[![EPISODE 1: Empty Module](http://img.youtube.com/vi/MV-Z23gSm-Q/0.jpg)](http://www.youtube.com/watch?v=MV-Z23gSm-Q)

It works in PowerShell Core, so no issues with this at all.

I think this is it for module creation in general. Next time we will look at CMDB integration. Our example will be a pseudo-CMDB running on Azure Table Service and we will be integrating to it as an example. Other options could be ServiceNow, for instance. It is really simple to integrate to it, but I don't have it in my hands so, we will use Azure tables instead.

See you next time, colleagues!