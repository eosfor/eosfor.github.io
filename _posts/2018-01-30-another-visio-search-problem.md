---
layout: post
title:  Another problem with search in Visio
categories:  ['PowerShell','Visio','Troubleshooting']
tags:  ['PowerShell','Visio','Troubleshooting']
date:  2018-01-30 05:08:27 +03:00
beginps: '<div class="syn_wrapper"><pre class="brush: powershell;">'
endps: '</pre></div>'
excerpt_separator: <!--more-->
---

Hello colleagues, this is another quick article regarding issues with search in Visio. It is somewhat extends the one of the previous [articles](visio2016-search-is-not-working)

So lets start ...
<!--more-->

First of all i need to say that i downloaded a stencil pack from [here](https://blog.sandro-pereira.com/2017/09/19/microsoft-integration-azure-and-much-more-stencils-pack-v26-for-visio/) and unfortunately search was not working for it. I tried the workaround i had found in the mentioned article, but it did not work either! It was really unexpected! I tried to reindex, move index to another folder. I even opened the Search Index DB with a hope to see something there. It was a waste of time, frankly.

So what i did then. I decided to see if there are any ETW providers for Windows Search. And there are a lot of them, so i used [Microsoft Message Analyzer](https://www.microsoft.com/en-us/download/details.aspx?id=44226) and started a tracing session as follows: 

![msganalizerproviders](/images/posts/msganalyzerproviders.png)

Then i put filter to the trace to see only messages with specific words like this ```contains "Event" and contains "Grid"``` and i managed to see queries Visio issues against Search Service!

![msganalizerquery](/images/posts/msganalyzerquery.png)

Great start, now it is a good time to try to execute this query. What we can do for that is to use PowerShell

{{ page.beginps }}
$sql = @"
SELECT "System.ItemPathDisplay", "Microsoft.Visio.MastersKeywords", "Microsoft.Visio.MastersDetails" FROM "SystemIndex" WHERE (SCOPE='file:') AND (("System.ItemPathDisplay" LIKE '%vss%') OR ("System.ItemPathDisplay" LIKE '%vsx%')) AND ((((CONTAINS ("Microsoft.Visio.MastersKeywords",'"event's"',1033) OR CONTAINS ("Microsoft.Visio.MastersDetails",'"event's"',1033))) OR ((CONTAINS ("Microsoft.Visio.MastersKeywords",'"evented"',1033) OR CONTAINS ("Microsoft.Visio.MastersDetails",'"evented"',1033))) OR ((CONTAINS ("Microsoft.Visio.MastersKeywords",'"eventing"',1033) OR CONTAINS ("Microsoft.Visio.MastersDetails",'"eventing"',1033))) OR ((CONTAINS ("Microsoft.Visio.MastersKeywords",'"events"',1033) OR CONTAINS ("Microsoft.Visio.MastersDetails",'"events"',1033))) OR ((CONTAINS ("Microsoft.Visio.MastersKeywords",'"events'"',1033) OR CONTAINS ("Microsoft.Visio.MastersDetails",'"events'"',1033))) OR ((CONTAINS ("Microsoft.Visio.MastersKeywords",'"event"',1033) OR CONTAINS ("Microsoft.Visio.MastersDetails",'"event"',1033)))) AND (((CONTAINS ("Microsoft.Visio.MastersKeywords",'"grid's"',1033) OR CONTAINS ("Microsoft.Visio.MastersDetails",'"grid's"',1033))) OR ((CONTAINS ("Microsoft.Visio.MastersKeywords",'"gridded"',1033) OR CONTAINS ("Microsoft.Visio.MastersDetails",'"gridded"',1033))) OR ((CONTAINS ("Microsoft.Visio.MastersKeywords",'"gridding"',1033) OR CONTAINS ("Microsoft.Visio.MastersDetails",'"gridding"',1033))) OR ((CONTAINS ("Microsoft.Visio.MastersKeywords",'"grids"',1033) OR CONTAINS ("Microsoft.Visio.MastersDetails",'"grids"',1033))) OR ((CONTAINS ("Microsoft.Visio.MastersKeywords",'"grids'"',1033) OR CONTAINS ("Microsoft.Visio.MastersDetails",'"grids'"',1033))) OR ((CONTAINS ("Microsoft.Visio.MastersKeywords",'"grid"',1033) OR CONTAINS ("Microsoft.Visio.MastersDetails",'"grid"',1033))))) ORDER BY "System.ItemPathDisplay"
"@

$Provider="Provider=Search.CollatorDSO;Extended Properties=’Application=Windows’;"
$adapter = new-object system.data.oledb.oleDBDataadapter -argument $sql, $Provider
$ds      = new-object System.Data.DataSet
if ($adapter.Fill($ds)) { $ds.Tables[0] }
{{ page.endps }}

And i got an answer, which means that files have been properly indexed and they exists in the Search Database.

![msganalizerquery2](/images/posts/msganalyzerquery2.png)

Luckily, i payed attention to the locale code in the query. And decided to look at the language of the stencil, and it appeared to be in Portugese! 

![portugesestencil](/images/posts/portugesestencil.png)

But all my drawings are in English!

![myfile](/images/posts/myfile.png)

So i just changed the language option on each of the stencil files i was interested in. Put them to a target location and copied/pasted as i did previously. And it WORKED!

To do this you need to open empty Visio drawing, then load a stencil, click ```edit``` on it, select ```Properties```, change the language and then save.

This is it!

Hope this helps someone!