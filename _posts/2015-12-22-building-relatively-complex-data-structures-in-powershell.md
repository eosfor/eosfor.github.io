---
layout: post
title: Building relatively complex data structures in Powershell
date: 2015-12-22 08:23:50.000000000 +03:00
categories: [Powershell, Azure]
tags: [Powershell, Azure]
---
Ok, let's continue our discussion about building an Azure asset database. In this short article I'm going to show a simple way of building a dynamic data structure which stores database alongside with its indexes.
Let's start from quick reminder of where we are. At the moment we have a variable, containing database of Azure based VMs. And we have a procedure of building indexes, based on [hash tables](https://en.wikipedia.org/wiki/Hash_table). How can we use it to search the DB in a cmdlet based way. I think that the best way is to use following semantics:

```powershell
Find-AzureDBObject -Database $DB -IPAddress 10.10.10.10
```

where *IPAddress* parameter is a name of an index. So here we have two problems. First - we need to store name of the index together with index itself (add index operation), and second, we need to enumerate indexes we have added (get index operation). In this case we going to get an option to dynamically add indexes to the structure. On the other hand we need this cmdlet to be able to dynamically add parameters to itself depending on the indexes of the DB.

In my opinion simplest way to achieve this is to build a structure of nested hash tables. For instance like this:

![img](/images/posts/oldposts/structure.png)

it simply contains database itself and links to a set of indexes. And as I mentioned before each index has links to the original database. Each of these structures is basically a hash table. For instance key described as following:

```powershell
$DB = @{Database = $ht; Index = @{}}<br />
```

where *$ht* is the original database, extracted from Azure. Second field will contain the list of indexes, like this

```powershell
$Index = @{'IPAddress' = $ipAddrIndex; StorageAccount = $storageAcctIndex; DriveFile = $storageFileIndex}
```

and as a result we build whole structure like this

```powershell
#create database</p>
$DB = @{Database = $ht; Index = @{'IPAddress' = $ipAddrIndex; StorageAccount = $storageAcctIndex; DriveFile = $storageFileIndex}}
```

This gives us an ability to add/remove indexes on the fly, use them and build cmdlets with dynamic parameters to easily use these indexes for search.
Next time we will look at how to use this in cmdlets
