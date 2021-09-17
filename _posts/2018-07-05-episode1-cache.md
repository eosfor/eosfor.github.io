---
layout: post
title:  '04. Episode 1: Cache, Singleton and Hashtables'
categories:  ['PowerShell']
tags:  ['PowerShell']
date:  2018-07-05 05:18:57 +03:00
beginps: '<div class="syn_wrapper"><pre class="brush: powershell;">'
endps: '</pre></div>'
---

Hi colleagues, we continue working on our Azure automation module. [Last time](https://eosfor.github.io/2018/Episode1-CMDB-integration/) we out together two components. One - to query data from Azure, and another one - to convert this data to format we need. Let's now focus on creating cache component.

First of all let's collect together some thoughts. Cache, typically, is a component, which is shared across multiple entities. It stores data to speed up access to it. So, we are trying to build something that has same two properties. First - it should be "only one", and second - it should store data and we need to be able to get this data from it. And we are going to make it as simple as possible.

<!--more-->

# "There can be only one"

Singleton, is the way to make something to be instantiated exactly once. Typically, this is done by:

- declaring all constructors of the class to be private; and
- providing a static method that returns a reference to the instance.

Here is how we can make it in PowerShell, using classes:

{{ page.beginps }}
class CMDBCache
{
    $cache = @{}
    static [CMDBCache] $Instance = [CMDBCache]::new()
    hidden CMDBCache()
    {
        $this.cache.DB = (Get-AzureTableData)
    }
}
{{ page.endps }}

We don't have "luxury" of having private constructor, so we declare it as ```hidden```. And we got the static member which returns the reference to the instance. In addition to this we have the [```hashtable```](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_hash_tables?view=powershell-6) ```$cashe```. This is basically a data structure which allows you to store key-value pairs. And it also has some more interesting features. For instance it is optimized for lookup by key, and we will use this feature shortly. But lets move on. Constructor basically initializes the key ```DB``` with the value, returned by ```Get-AzureTableData```, so at the moment we want to get the value of the DB key, it is initialized with our data.

# Using our cache class

So how we can use it? It is simple:

{{ page.beginps }}
function Get-AzureCMDBData {
    [CmdletBinding()]
    param (
        [switch]$BU,
        [switch]$DEPT
    )

    process {
        if($BU.IsPresent) {
            [CMDBCache]::Instance.cache.DB | ? PartitionKey -EQ "BU"
            return
        }
        if($DEPT.IsPresent) {
            [CMDBCache]::Instance.cache.DB | ? PartitionKey -EQ "DEPT"
            return
        }
        [CMDBCache]::Instance.cache.DB
    }
}
{{ page.endps }}

What we do here is, we basically the function which takes two switch-parameters. They identify which part of our dataset to return. If none is set it will return the whole dataset. As you can see, here is the syntax we use here: ```[CMDBCache]::Instance.cache.DB```. Basically, ```::``` means that we accessing a static member. So in this case, syntactically, we use static member ```Instance``` to get to the ```cache.DB```. This, on first call, will first instantiate and initialize an object with a constructor, and on all subsequent calls will simply return our data, without calling ```Get-AzureTableData``` again.

Here you can find some more information :)

[![EPISODE 1: Caching Azure Storage Table calls](http://img.youtube.com/vi/0mYWMPtjzuM/0.jpg)](http://www.youtube.com/watch?v=0mYWMPtjzuM)
