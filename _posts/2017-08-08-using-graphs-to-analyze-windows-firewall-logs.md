---
layout: post
title: Using graphs to analyze Windows Firewall logs
date: 2017-08-08 00:53:57.000000000 +03:00
type: post
categories: [PowerShell, PSQuickGraph]
tags: [PowerShell, PSQuickGraph]
beginps: '<div class="syn_wrapper"><pre class="brush: powershell;">'
endps: '</pre></div>'
---
Hello colleagues, lets talk about how we can use graphs to look inside of communications happening in our environments in an easy way. First of all we need to have some data to analyze. Lets gather some. It is pretty simple - just use [this article](https://technet.microsoft.com/en-us/library/cc947815(v=ws.10).aspx) and enable Windows Firewall Logging. I usually put the logs into a separate folder, just for easy access. Here is how it looks like on my system:

![img]({{ site.baseurl }}/images/posts/oldposts/fwlog.png)

As you can see, it is structured, so it is good idea to parse it as objects. Lets do it, and it may look like the following:


{{ page.beginps }}
$f = gc "C:\Temp\pfirewall_public.log"
$regex = '^(?\<datetime\>\d{4,4}-\d{2,2}-\d{2,2}\s\d{2}:\d{2}:\d{2})\s(?\<action\>\w+)\s(?\<protocol\>\w+)\s(?\<srcip\>\b(?:\d{1,3}\.){3}\d{1,3}\b)\s(?\<dstip\>\b(?:\d{1,3}\.){3}\d{1,3}\b)\s(?\<srcport\>\d{1,5})\s(?\<dstport\>\d{1,5})\s(?\<size\>\d+|-)\s(?\<tcpflags\>\d+|-)\s(?\<tcpsyn\>\d+|-)\s(?\<tcpack\>\d+|-)\s(?\<tcpwin\>\d+|-)\s(?\<icmptype\>\d+|-)\s(?\<icmpcode\>\d+|-)\s(?\<info\>\d+|-)\s(?\<path\>.+)$'

$log =
$f | % {
    $_ -match $regex | Out-Null
    if ($Matches) {
    [PSCustomObject]@{
        action   = $Matches.action
        srcip    = [ipaddress]$Matches.srcip
        dstport  = $Matches.dstport
        tcpflags = $Matches.tcpflags
        dstip    = [ipaddress]$Matches.dstip
        info     = $Matches.info
        size     = $Matches.size
        protocol = $Matches.protocol
        tcpack   = $Matches.tcpac
        srcport  = $Matches.srcport
        tcpsyn   = $Matches.tcpsyn
        datetime = [datetime]$Matches.datetime
        icmptype = $Matches.icmptype
        tcpwin   = $Matches.tcpwin
        icmpcode = $Matches.icmpcode
        path     = $Matches.path
    }
    }
} 
{{ page.endps }}

The **$regex** variable here contains a long regular expression which is going to parse our file onto objects - one per line. We use **-match** operator in the pipeline to apply this expression to each line of the file and suppress output by piping it to **Out-Null**. This is not the fastest way of parsing files but to me one of the easiest ones. If the there is a match **$Matches** variable gets populated. Here we want to do the trick. First we fill in the hashtable with the fields we would like to put into our new object and then convert this hashtable to an object. One thing to pay attention to is we convert datetime field into [datetime] type to be able to use filtering and sorting capabilities later on. The same we do with ip addresses. So at the end we've got objects and they look like this:

![Img]({{ site.baseurl }}/images/posts/oldposts/fwlog3.png)

Looks great so far, but what is next? First of all objects we've got are just edges of our graph. So what we can do now to convert the set of edges to a set of vertices along with their edges? This is really easy, lets just add them

{{ page.beginps }}
$g = new-graph -Type BidirectionalGraph
$log | ? {$_.srcip -and $_.dstip} | % {
    Add-Edge -From $_.srcip -To $_.dstip -Graph $g | out-null
}
{{ page.endps }}

So here we create a graph and add vertices. Source and destination IPs being converted into string representations and added to the graph, and the library itself takes care about duplicated entries. So at the end we have a $g variable containing the graph. Now we can easily display it by issuing

{{ page.beginps }}
Show-GraphLayout -Graph $g
{{ page.endps }}

which will display something like this

![Img]({{ site.baseurl }}/images/posts/oldposts/fwlog4.png)

It does not look beautiful for me as it basically shows only a single log from my own laptop. But if we had multiple logs from some environment we would be able to see communications happened inside and outside the environment.

What else can we do here? For example we can try to filter the set of log data we parsed and display the smaller subset of data, for instance like this

{{ page.beginps }}
$d = ($log | sort datetime -Descending | select -First 1).datetime.addhours(-1)
$twoHrsLog = $log.Where({$_.datetime -gt $d})
$g1 = new-graph -Type BidirectionalGraph
$twoHrsLog | ? {$_.srcip -and $_.dstip} | % {
    Add-Edge -From $_.srcip -To $_.dstip -Graph $g1 | out-null
}
Show-GraphLayout -Graph $g1
{{ page.endps }}

where we just filter the log to see just communications happened for the last hour. We could also filter by IPs or by degree of out or in edges

{{ page.beginps }}
$g2 = new-graph -Type BidirectionalGraph
$x = $g.Vertices.Where({$g.OutDegree($_) -gt 0})
$x | where {$_ -ne '192.168.0.107'} | % {$e = $g.InEdges($_); if ($e) {$e | % {add-edge -from $_.source -to $_.target -Graph $g2}}}
$x | where {$_ -ne '192.168.0.107'} | % {$e = $g.OutEdges($_); if ($e) {$e | % {add-edge -from $_.source -to $_.target -Graph $g2}}}

Show-GraphLayout -Graph $g2
{{ page.endps }}

Complete set of commands is below

{% gist f59298b0ab6ceea170df3d69efb4787f winFWAnalizeGist.ps1%}