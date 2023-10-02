---
layout: post
title: Enhancing VM Search in Azure with PowerShell
date: 2015-12-17 10:35:09.000000000 +03:00
tags: [PowerShell, Azure]
---

In this walkthrough, we'll explore various functionalities of PowerShell, including dynamic parameters, parallel execution of script blocks, and index building for efficient searches, among other features.

Let's dive in by understanding the problem at hand. Often, there's a need to search VMs not only by their names but also by their internal IP addresses or the names of their disks. However, the absence of an out-of-the-box API for this task led me to develop a custom tool. The goals were outlined as follows:

- Speedily extract data from Azure using parallel threads.
- Avoid multiple lookups in the VM array by employing an index.
- Ensure the flexibility to add indexes as needed, with search commands adapting dynamically.
- Prioritize the use of standard, documented REST calls.
- Utilize classic REST calls, given that nearly all VMs are V1.

For a quick glance at the code, [check out the samples here](https://github.com/eosfor/AzureSearch).

## Extracting Data

Our environment comprises more than ten subscriptions, a number that's slated to grow. The lookup procedure should seamlessly span all existing subscriptions and automatically discover new ones as they emerge. The pragmatic approach I identified was to locally store all data extracted from Azure in an array. Initial trials using the traditional PowerShell Azure cmdlets like `Get-AzureVM` proved to be slow, especially for a single subscription. An attempt to run this cmdlet in parallel using the [PSParallel](https://github.com/powercode/PSParallel) module was unfruitful, as some cmdlets within the MS Azure module seemed unprepared for parallel execution. The issues suspected to emanate from the `Select-AzureSubscription` part resulted in unpredictable failures and peculiar error messages. After a period of experimentation, I opted to switch gears to the Azure REST API for data extraction. Although this shift made it unfeasible to pass the returned objects to other Azure cmdlets, the upside was a significantly quicker data retrieval. The way to circumvent this limitation remains a subject for future exploration.

The first step was to list cloud services in parallel using [this call](https://msdn.microsoft.com/en-us/library/azure/ee460781.aspx?tduid=(7e6af0dae94cd99be610744b05b54dd4)(256380)(2459594)(XdSn0e3h3.k-wrFcUJSbpSAsQV7.MdbuBQ)()).

```powershell
$sub = Get-AzureSubscription
$h = $sub.SubscriptionId | Invoke-Parallel { Invoke-RestMethod -uri https://management.core.windows.net/$_/services/hostedservices -Method GET -Headers $headers }
```

This method was efficient and speedy. Following that, I executed [get deployment](https://msdn.microsoft.com/en-us/library/azure/ee460804.aspx) to extract VM data and store it into a hashtable using the service name as a key. Upon completion, a list of all VMs across all subscriptions was at my disposal.

```powershell
$ht = [hashtable]::Synchronized(@{})
$h.HostedServices.HostedService | skip-null | invoke-parallel {
    #declare function inside of the script block
    function xmlToObject {
        param($o)
        $h = @{}
        $o  | gm -MemberType Property | % {
            $prop = $_.name
            if ($o.'$prop' -is 'System.Xml.XmlElement') {
                $h[$prop] = xmlToObject $o.'$prop'
            }
            else { $h[$prop] = $o.'$prop' }
        }
        $h
    }
    try {
        $curr = $_
        $subId = ([uri]$curr.Url).Segments[1] -replace '\/', ''
        $d = Invoke-RestMethod -uri "https://management.core.windows.net/$subId/services/hostedservices/$($curr.ServiceName)/deploymentslots/production" -Method GET -Headers $headers
        if ($d) {
            $ri1 = $d.Deployment.RoleInstanceList.RoleInstance | % { xmlToObject $_ }
            $ro1 = $d.Deployment.RoleList.Role | % { xmlToObject $_  }
            $ht[$curr.ServiceName] = $ri1 | % {$c = $_; $r = $c.'RoleName'; $x = $ro1 | where {$_.'RoleName' -eq $r}; $x.remove('RoleName'); [pscustomobject]($_ + $x) }
        }
        else { $ht[$curr.ServiceName] = $null }
    }
    catch { $data = 1 }
}
```

A couple of points to note here. Firstly, the data returned by the call is formatted peculiarly. VM information is stored in

 two different places: in the `RoleInstanceList` subtree and the `RoleList` subtree. To streamline the subsequent index creation, I converted the mentioned XML subtrees to hashtables, merged them, and produced a `pscustomobject`. This new object was then added to a hashtable under the appropriate cloud service. Secondly, the conversion was executed by the `xmlToObject` function, which had to be explicitly added to each "thread". Hence, the script block to run comprises the definition of the `xmlToObject` function alongside the REST method invocation.

In essence, this script block extracts all deployments from the Cloud Service, retrieves machine details from the results, creates objects for each machine, and adds these objects to a hashtable under a Cloud Service Key. Consequently, we have a list of VMs indexed by Cloud Service Name.

## Building Indexes

At this juncture, all the necessary data is at our fingertips. However, the need for a sudden lookup may arise, and without proper indexing, this process could be time-consuming, especially with multiple lookups. A straightforward solution is to build additional indexes using PowerShell hashtables:

```powershell
# prepare indexes
$ipAddrIndex = @{}
$storageAcctIndex = @{}
$storageFileIndex = @{}
#build indexes
$ht.Keys | % {
    $ht[$_] | % {
        if ($_.ipaddress) {$ipAddrIndex[$_.ipaddress] = $ipAddrIndex[$_.ipaddress] + (, $_)}
        else {$ipAddrIndex['noip'] = $ipAddrIndex['noip'] + (, $_)}
    }
}
$ht.Keys | % {
    $ht[$_] | % {
        if ($_.OSVirtualHardDisk) {$url = [uri]$_.OSVirtualHardDisk.MediaLink; $storageAcctIndex[$url.host] = $storageAcctIndex[$url.host] + (, $_)}
        else {$storageAcctIndex['noDisk'] = $storageAcctIndex['noDisk'] + (, $_)}
    }
}
$ht.Keys | % {
    $ht[$_] | % {
        $curr = $_
        if ($_.OSVirtualHardDisk) {$url = [uri]$_.OSVirtualHardDisk.MediaLink; $storageFileIndex[$url.Segments[-1]] = $storageFileIndex[$url.Segments[-1]] + (, $_)}
        else {$storageFileIndex['noDrive'] = $storageFileIndex['noDrive'] + (, $_)}
        if ($_.DataVirtualHardDisks) {
            $_.DataVirtualHardDisks.DataVirtualHardDisk | % {
                $url = [uri]$_.MediaLink; $storageFileIndex[$url.Segments[-1]] = $storageFileIndex[$url.Segments[-1]] + (, $curr)
            }
        }
    }
}
```

The task here involves traversing the array of VMs, extracting a value of the property we are interested in, and using it as a key in a hashtable. The value for this key is the object itself. For instance, a basic index could be created by the private IP address of the VMs. In this scenario, we run through objects in the original hashtable, extract the IP address of the machine, and create a record in a new hashtable with this IP as a key and the current object as the value.

```powershell
$ht.Keys | % {
    $ht[$_] | % {
        if ($_.ipaddress) {$ipAddrIndex[$_.ipaddress] = $ipAddrIndex[$_.ipaddress] + (, $_)}
        else {$ipAddrIndex['noip'] = $ipAddrIndex['noip'] + (, $_)}
    }
}
```

Now, all our data is housed in a hashtable, along with a couple of indexes. In my case, they are by IP address, by storage account, and by VHD file names of the VMs. With these indexes, we can search for a VM using just the semantics of the hashtable as shown below. It returns all the VMs across all subscriptions with a private IP address of 10.10.10.10.

```powershell
$ipAddrIndex['10.10.10.10']
```

In the next article, we'll explore how to construct a cmdlet that dynamically builds its parameter set based on the names of indexes to perform lookups on those indexes.