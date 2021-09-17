---
layout: post
title: Quick and dirty way of checking links
date: 2015-03-13 09:34:49.000000000 +03:00
categories: [PowerShell]
tags: [PowerShell]
---
So ho can we run through the site and check if links are ok on it? I mean if it is possible to automate this using powershell? Another thing here is how to do this fast, better in parallel threads. The problem here is that there is no threads in Powershell unfortunately. There are only Powershell Jobs, but they are quite heavy. So what to do and how to achieve this? I've taken the idea from [this post](http://learn-powershell.net/2012/05/13/using-background-runspaces-instead-of-psjobs-for-better-performance/). In general i run [Invoke-WebRequest](https://technet.microsoft.com/en-us/library/hh849901.aspx) cmdlet as a scriptblock in separate parallel runspaces. [Here](https://github.com/eosfor/Powershell-Tips/blob/master/TestSite.ps1) you can have a look. My contribution in this is not very big. I've just updated the scriptblock running as part of the thread like below:

<pre class="brush: powershell;">
$ScriptBlock = {
    Param ($lnk)
    $progressOld = $ProgressPreference
    $ProgressPreference = 'SilentlyContinue'
    $obj = new-object psobject -prop @{Name = $lnk;
                                       Status = (Invoke-WebRequest -Uri $lnk -DisableKeep).statuscode
    }
    $ProgressPreference = $progressOld
    $obj
}
</pre>

Also i've added some checks to skip repeating links and links that does not start from "http*"
