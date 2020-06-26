---
layout: post
title: "How to attach HTTPS custom domain to a storage account-based static website"
categories:  ['PowerShell','WSL', 'Windows']
tags:  ['PowerShell','WSL', 'Windows']
date:  2020-06-25 21:45:14 -07:00
beginps: '<div class="syn_wrapper"><pre class="brush: powershell;">'
endps: '</pre></div>'
images: "/images/posts"
excerpt_separator: <!--more-->
---

Well, in their [documentation](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blob-static-website#mapping-a-custom-domain-to-a-static-website-url) MSFT says that:

> To enable HTTPS, you'll have to use Azure CDN because Azure Storage does not yet natively support HTTPS with custom domains

However, I've found a quick trick to make it possible. Well, actually, [they also did it](https://azure.microsoft.com/en-us/services/app-service/static/) :). But I decided to make this post anyway.

So in general, there is a [proxy functionality](https://docs.microsoft.com/en-us/azure/azure-functions/functions-proxies) on top of Azure Functions. This allows us to put a proxy in front of a storage account and redirect all calls to a function to the storage account static website URL. And then you can configure a custom domain for a Function as usual. Basically all you need to do is to:

- create a storage account and enable it for static sites
- create a windows Azure function. I used Consumption plan for my test
- configure the proxy functionality to redirect requests from a function to the storage account static web app endpoint. It can be dove via portal
- attach custom domain to the function
- generate and attach certificate to the function

That is basically it.
