---
layout: post
title: How to search for an object in Azure
date: 2015-11-02 19:35:54.000000000 +03:00
categories: [PowerShell, Azure]
tags: [PowerShell, Azure]
---

In our environment we have more than one subscription. Even more than five. And this amount grows constantly. What we do with all of this is we support VMs and stuff there. One of the issues is that users usually do not know name of Azure Service which is used to host their VMs, and it so happens that they do not know even their Subscription Name or Subscription ID. To bring up and control their VMs and environments the use some “middleware” which hides all of this info from them. So they usually come to us and say: “here you are a list of VMs we have problems with, please have a look and fix”. Sometimes such list contains not only VMs but storage accounts along with VMs etc. Unfortunately classic azure cmdlets does not provide an option to search for objects in the cloud by their name. On the other hand new Azure Management portal does. It can find objects by name. So I decided to do a little hack end use this API in order to simplify our live.

<!--more-->

## Solution

I have to admit that I’m not a developer – I’m infrastructure guy. I’m quite far from all of that REST stuff so my definitions might be not really accurate. I’ll try to describe some steps I took to find and use this “hidden” API, as far as I did not find any documentation regarding it. Actually, it was pretty easy to find the actual  REST call using [Fiddler](https://www.telerik.com/fiddler). If it does not wok for you now, as it was working couple of weeks ago, you can use Message Analyzer for the same. They both do the trick. So I reproduced the search query in new portal in browser and captured the REST call. It was a call to URI **https://management.azure.com/api/invoke**. The interesting thing was a **x-ms-path-query**’ header containing the following:

>/resources?api-version=2014-04-01-preview;$filter=(subscriptionId eq 'SubID' or subscriptionId eq 'SubID' or subscriptionId eq 'SubID' or subscriptionId eq 'SubID' or subscriptionId eq 'SubID' or subscriptionId eq 'SubID' or subscriptionId eq 'SubID' or subscriptionId eq 'SubID' or subscriptionId eq 'SubID') and ((substringof('SomeName', name) or substringof('SomeName', resourcegroup)))

I was trying to google for pieces of it but did not find anything, except that it looked like a subset of [Azure AD Graph API](https://msdn.microsoft.com/en-us/library/azure/dn727074.aspx). Well, that was something, so next step was just to try to call it via PowerShell. But how to authenticate over Azure and put correct "authenticator" into HTTP request? [This](http://blogs.technet.com/b/keithmayer/archive/2014/12/30/authenticating-to-the-azure-service-management-rest-api-using-azure-active-directory-via-powershell-list-azure-administrators.aspx) helped me a lot.

After I got the Auth Header I was able to run a REST query captured at the very beginning. REST call returns a set of JSON objects, i parse them for later use and they look like below

```console
PS C:\&gt; Get-AzureObject -Name ACDB2-AG -RawOutput -All
SubscriptionID : 35882c3a-d01e-479c-a46e-67903c9cae39
type : Microsoft.ClassicCompute/virtualMachines
location : eastus
ObjectName : VMNAME-1
ResourceGroup : ResourceGroup1
<p>SubscriptionID : 49a40aa2-5cb7-40ca-abd9-9de732c93a0d
type : Microsoft.ClassicCompute/virtualMachines
location : eastus
ObjectName : VMNAME-1
ResourceGroup : ResourceGroup2
```

So whole solution works as follows:

- Runs a query as is against all registered subscriptions and names provided
- Parses the results and removes all that is not required according to parameters, for example, remove all results except VMs
- For each object from step 3 uses classic cmdlets to receive actual objects using Subscription, ObjectName and resource group returned by the original search query

{% gist 9c51ea9ec66c114ce947 getAzureObject.ps1 %}

I have to say that this piece of code uses some proxy functions that I wrote to hide Subscription context switching. But I'll describe them next time.
