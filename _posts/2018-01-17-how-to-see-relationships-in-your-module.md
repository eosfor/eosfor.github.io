---
layout: post
title:  How to see relationships in your module
categories:  ['PowerShell']
tags:  ['PowerShell']
date:  2018-01-17 05:18:26 +03:00
beginps: '<div class="syn_wrapper"><pre class="brush: powershell;">'
endps: '</pre></div>'
excerpt_separator: <!--more-->
---

Hello colleagues, long ago i blogged about a quick and dirty way to see the graph your function calls in the powershell script. Now it is time to see the graph for such calls or dependencies in your PowerShell module.
<!--more-->

I need to mention here that this only works with pure PowerShell based modules.

For that what we need is to install [PSQuickGraph module](https://www.powershellgallery.com/packages/PSQuickGraph/1.1) from Powershell Gallery. Next what we need to do is to run the following script, with small modifications:

{% gist e66b1bb2c4d7fa5279dc1ffbcfd6f205 showModuleGraph.ps1 %}

The first thing we need to do is to change ```"C:\Repo\Work\Tools\*.ps1"``` to a proper module path.
Next thing is the last line of the script, there you need to put a correct path.

What this script does. First of all it shows a simple GUI with a graph and you can play around with this GUI. And in the last line it exports the graph to a DOT format, so you can use graphviz tool to generate a pretty picture of it, if you'd like.

This is how it looks like for me. I made it smaller just tou give you some understanding of how it looks like

![img]({{ site.baseurl }}/images/posts/moduleStructure.png)