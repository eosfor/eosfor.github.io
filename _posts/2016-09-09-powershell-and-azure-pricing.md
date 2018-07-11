---
layout: post
title: PowerShell and Azure pricing
date: 2016-09-09 14:42:10.000000000 +03:00
categories: [PowerShell, Azure]
tags: [PowerShell, Azure]
---
Sometimes it is difficult to calculate prices for Azure. Especially when you have a lot of machines. What to do? It is easy with PowerShell:

<pre class="brush: powershell;">
$request = Invoke-WebRequest -Uri https://azure.microsoft.com/api/v1/pricing/virtual-machines/calculator/?culture=en-us
$offers = $request.Content | ConvertFrom-Json
$offers.offers.'standard-d11v2'.prices | select -Property 'us-east'
</pre>

Enjoy!
