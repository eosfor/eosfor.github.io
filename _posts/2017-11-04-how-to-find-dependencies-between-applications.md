---
layout: post
title: How to find dependencies between applications
date: 2017-11-04 16:10:28.000000000 +03:00
categories: [PowerShell, PSQuickGraph]
tags: [PowerShell, PSQuickGraph]
---
Hello colleagues. As part of migrations to the cloud most common question is - how to map existing application landscape? How to find all relations of the application being moved? How to see which systems it communicates to?</p>
Actually I already blogged about this some time ago but this time example is a bit more clear. We need to do the following steps to graph our environment:

- Enable and collect Windows Firewall logs  for all systems in the environment</li>
- Download and install [PSQuickGraph](https://www.powershellgallery.com/packages/PSQuickGraph/1.1)
- Analyze the logs


For the second step here is the simplest example
{% gist a29f53052c7bdfc98339e630149cccdb %}

And here is what we can get out of it.

![Img](/images/posts/oldposts/apprelgraph.png)
