---
layout: post
title:  '03. Episode 1: PowerShell, Azure Tables, CMDB integration'
categories:  ['PowerShell', 'Azure']
tags:  ['PowerShell', 'Azure']
date:  2018-07-04 05:00:22 +03:00
beginps: '<div class="syn_wrapper"><pre class="brush: powershell;">'
endps: '</pre></div>'
---

Hello colleagues, today we are going to look at CMDB integration components, what benefit we going to get from them, how to make them and what to use them for.

<!--more-->

# General idea

Typically, from the management perspective, the cornerstone for all environments I've seen so far were ITSM model and [CMDB](https://en.wikipedia.org/wiki/Configuration_management_database). These components is the most common way, and, perhaps, the only way to be able to manage big and diverse infrastructures. My personal opinion is that even, when you have something small, some small environment or an application you should build and maintain some kind of high level CMDB. It does not need to be big, but it should contain some key information you may need to identify environments, owners, stakeholders etc. In our example we assume we have some CMDB running somewhere in the Cloud, and we want to query it. For now we will focus on querying organizational structure of our company.

Why we need this? Well, it is difficult to explain in simple words. There are a lot of benefits. The main one, perhaps, is it allows you to easily find relationships between your assets and respective teams, stakeholders and beneficiaries. Which, in turn, allows you to easily identify upstream and downstream systems and affected stakeholders in case of failures or in other cases, for example, migrations.

For our case, however, lets focus on Naming and Org. Structure first. We want our automation to generate names for Azure resources. For that we want it to query data from CMDB. On the other hand we may want to replace our CMDB provider sometime in future, so we need to make our module in a way, that we can easily do this. This will allow us to abstract other functionality we will have in our module, from CMDB details. So the goals are:

- Integrate with CMDB
- Make integration piece replaceable
- Abstract internal functionality from CMDB connector implementation

And of course we want to see how to do it in PowerShell. So let's start.

# General design

Let's first build our design first, which can cover all of these goals. Here is a big picture of it.

First of all, we going to have some, say, "internal" components which will use CMDB services. On this picture one of such components is represented by "CMDB cmdlet" block. Any other component like it, should be abstracted from CMDB interface, which represented as "replaceable part" block on the diagram. It understands CMDB provider internals, can query it, interpret results and so on. In general, this piece of software serves as a bridge between "internal" components and 3rd party CMDB. And we want to make this "bridge" easily replaceable in our PowerShell code.

![design](/images/posts/2018-07-04-Episode1-CMDB-integration/design.png)

## CMDB Aware cmdlet

This component is the CMDB connector itself. It knows the details of exactly one CMDB. It's task to talk to CMDB, request data using CMDB API and terminology. And this component is not aware of any thing except CMDB.

## Interface cmdlet

This component abstracts CMDB interface with the interface expected by "internal" components. It is kind of an adapter. On one hand it understands the interface provided by "CMDB Aware" component, and on other hand it exposes another interface, which is known by upstream components. So in general, when we decide to move out of existing CMDB to something else, we can do it by implementing another pair of these connector/adapter components. We need to make sure, however, that this adapter always provides the same interface.

In common programming languages, like C# this requirement is fulfilled with something called "[Interfaces](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/interface)" or "[Abstract classes](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/abstract)". However, even with classes, as far as I know, there is no simple way to define an interface, especially for cmdlet in PowerShell. [There are some tricks](https://youtu.be/05Jqr7GophU), however I don't really like using them for production, because they are too hardcore and apply a lot of complexity. And I personally believe that complexity is bad, especially with automation. So in our case we will just make them as cmdlets and assume, that in case we need to implement a new one, we will just implement the same set of parameters. But, unfortunately, we don't have a strong option to enforce this interface.

## Cache component

When talking to remote systems running somewhere in the Cloud, the common problem is latency. Well, it is not problem in general, or, probably, better to say not, that big issue. But in our case we want it to be faster for a few reasons. First of all we are going to use them later as part of our auto-completion for parameter values, and this is critical. People do not want to wait few seconds while your web request completes. And if they want to call some cmdlet more than once, it is good idea to cache some data to avoid this "slow" behavior. So in our example we want to cache CMDB data to speed things up and to see how to make your own simple in-memory cache with standard .NET and PowerShell components. It is really easy, I should say.

# Working with Azure (Tables) - Implementing CMDB Aware component

So, as I said previously we will 'mimic' CMDB with Azure Tables. Why I decided to do this? Well, I wanted to have something running in the Cloud which would be "queryable", available via Powershell cmdlets or .NET in some simple manner. So, the [Azure Table Storage](https://docs.microsoft.com/en-us/azure/storage/tables/table-storage-overview) meets all of these. So, we can imagine that this is some kind of an 3rd party CMDB available via REST API and which has some .NET SDK provided. Another goal I set for myself is to show how easy it is to get all you need just by using console. I mean, by just using [Get-Member](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-member?view=powershell-6) cmdlet and some knowledge of programming you can quickly find all you need to work with such things. So, if you don't know which cmdlets are there for a specific task or which classes to use - go for [Get-Member](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-member?view=powershell-6) and it will give you all answers.

Here is how to do this live :) :

[![EPISODE 1: Empty Module](http://img.youtube.com/vi/gZn3VrhlPJM/0.jpg)](http://www.youtube.com/watch?v=gZn3VrhlPJM)

Basically, our "CMDB Aware component" consists of two parts. First, is a part which talks directly to our CMDB in the Cloud. Second part is the one which converts all the stuff we got from the Cloud into something we can easily reuse in PowerShell later - arrays of PowerShell objects. On this iteration we implement the first part this way:

{{ page.beginps }}
function Get-AzureTableData {
    [CmdletBinding()]
    param (
        $RGname = "trainingRG01",
        $StorageAccountName = "somestorage",
        $TableName = "CMDB"
    )

    begin {
        $storageAcct = Get-AzureRmStorageAccount -ResourceGroupName $RGname -Name $StorageAccountName
        $tbl = Get-AzureStorageTable -Name $TableName -Context $storageAcct.Context
    }

    process {
        $q1 = [Microsoft.WindowsAzure.Storage.Table.TableQuery]::new()
        $t1 = [Microsoft.WindowsAzure.Storage.Table.TableContinuationToken]::new()

        $res = $tbl.CloudTable.ExecuteQuerySegmentedAsync($q1, $t1).Result

        Convert-AzureData -azureRows $res
    }
}
{{ page.endps }}

As you see this is an PowerShell [Advanced Function](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_advanced?view=powershell-6) with CmdletBinding attribute, which allows it to be used in a PowerShell pipeline and have some additional common parameters automatically. Commonly, when you need to talk to some Azure-based resource, you want to provide couple of pieces of information. First of all it is a Resource Group name, and the Resource name itself. In our case it is name of the Storage Account. And we also need to know which table to query. So we have tree parameters here. Next thing we need is we need to get some objects, which we will use to work with the Storage. We do this in the ```begin``` block, because we don't want this piece to be executed each time something comes to this cmdlet over a pipeline. We request for the Table, which is a part of the Storage Account. So far it seems logical. Next thing is the query itself. It is in the process block. We call it each time something comes over a pipeline, but may be we may need to change this behavior, i don't know yet :). So, after some experiments we made, and you can find them in the video, we figured out that there is a function ```ExecuteQuerySegmentedAsync``` under the ```CloudTable``` member, which, seems like, fits our needs. Frankly, I checked on the Internet it after I made this test, and found docs for it [here](https://docs.microsoft.com/en-us/dotnet/api/microsoft.windowsazure.storage.table.cloudtable.executequerysegmentedasync?view=azure-dotnet). It is exactly what we need to issue a query. We can even extend our function and provide more sophisticated query and make some parameters to be able to give users an option to provide thier own queries for Azure Tables, but for this example let's keep it simple. So, this function takes two parameters, and we don't need to supply any initial values for them, we just need to instantiate them. Previously we only had a [New-Object](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/new-object?view=powershell-6) cmdlet for creating .NET objects out of .NET Type name. But at some point we got a new syntax using ```::new()``` function. It is a static function which represents a constructor. So, in our case, we simply call this function to create these objects, and then simply pass these objects to a ```ExecuteQuerySegmentedAsync``` method. PowerShell then takes care of all the *Async* related handling. Not sure how it does this, frankly :). Will try to look for this info later. But for the time being we get an answer from Azure Table in our hands.

Next thing we need to do is to convert this data into something we can reuse later. For that we develop another function:

{{ page.beginps }}
function Convert-AzureData {
    [CmdletBinding()]
    param (
        $azureRows
    )

    begin {
    }

    process {
        # iterathe through all rows
        $azureRows | % {
            #save a reference to a current row
            $r = $_

            # start another iteration over all stuff stored under Properties property
            $r.Properties | % {
                #save the current property
                $currentProp = $_
                #instantiate the temporary hashtable
                $o = [hashtable]::new()
                #iterate through Dictionary using Keys
                $currentProp.Keys | % {
                    #for each Key take it's name and the value and put them into a temporary hashtable
                    $o[$_] = ($currentProp)[$_].PropertyAsObject
                }
                # add some more data to hashtable
                $o["PartitionKey"] = $r.PartitionKey
                $o["RowKey"] = $r.RowKey
                $o["Timestamp"] = $r.Timestamp
                # convert hashtable to the powershell object and return it to a pipeline
                [pscustomobject]$o
            }
        }
    }
}
{{ page.endps }}

What it does is it just iterates through all objects provided to it as a parameter, extracts ```Properties``` property and iterated through all of them. Each property is basically a Dictionary, having a name of a field as a Key of the Dictionary and a value, well, as a value. However, values are stored as instances of a [```DynamicTableEntity ``` class](https://docs.microsoft.com/en-us/dotnet/api/microsoft.windowsazure.storage.table.dynamictableentity?view=azure-dotnet), but we want to extract actual values.  So we iterate through all Keys, extract their actual values and  put all of these into a temporary ```hashtable```. 

Hash tables in PowerShell have two really nice features. First of all you can add key/value pairs to it just at the moment you assign something, like ```$h.newkey = "somth"```, and this key ```newkey``` will be dynamically added. This is really useful. Another feature is that you can quickly convert a hash table into ```pscustompobject``` by using cast syntax. And when having ```pscustompobjects``` you can easily reuse them later in iterations, filters, searches and so on. Even more, if this kind of an array is big, say thousands of objects, you can speedup querying it by building your own index on top of it, but, well, we will look at this a bit later. 

So now we just extract properties by iterating trough received objects, and "converting" the dataset we've got into an array of ```pscustompobjects```. As an output ```Convert-AzureData``` generates a stream of objects thrown at the pipeline. And at the end they all will be returned by ```Get-AzureTableData```.

Well, this is basically it. I hope I managed to explain some complexities you may face looking though this video, but I encourage you to have a loot at it and as your questions or provide your suggestions.

Next time we will look at cache component, and we will use Singleton pattern and PowerShell Classes to build it

Thanks, colleagues! See you next time!