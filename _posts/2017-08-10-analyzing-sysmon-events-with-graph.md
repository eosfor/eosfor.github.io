---
layout: post
title: Analyzing sysmon events with graph
date: 2017-08-10 13:37:07.000000000 +03:00
categories: [PowerShell, PSQuickGraph]
tags: [PowerShell, PSQuickGraph]
---
Hello colleagues, this is an example I promised answering [this](https://twitter.com/cyb3rops/status/895024725730238464) tweet. I used this [sysmon config](https://github.com/SwiftOnSecurity/sysmon-config) to capture activities happening on my system. Unfortunately it did not capture a lot of network-related activities, perhaps I need to change it to extend network-level filters. But on the other hand it captured a lot of process level activities, so in this example i'd like to try to graph process creation events.
So first thing to do in this case is create a graph object

<pre class="brush: powershell;">
$g = New-Graph -Type BidirectionalGraph
</pre>

And now we can fill in the graph with some data right from the event log. It may take few seconds until all events are processed

<pre class="brush: powershell;">
Get-WinEvent -LogName Microsoft-Windows-Sysmon/Operational | ? {$_.id -eq 1} | 
    % { if ($_.properties[3]) `
            {Add-Edge -From $_.Properties[-2].value -To $_.properties[3].value -Graph $g}} | 
    Out-Null
</pre>

And then we just display the graph

<pre class="brush: powershell;">
Show-GraphLayout -Graph $g
</pre>

This is how it looks like

[![img](http://img.youtube.com/vi/LuRo8GEwp1w/0.jpg)](http://www.youtube.com/watch?v=LuRo8GEwp1w)
