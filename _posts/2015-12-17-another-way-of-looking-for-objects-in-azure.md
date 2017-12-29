---
layout: post
title: Another way to search objects in Azure
date: 2015-12-17 10:35:09.000000000 +03:00
categories: [Powershell, Azure]
tags: [Powershell, Azure]
---
##Overview.
In this article we are going to have a look at number of things in PowerShell. We are going to try dynamic parameters, parallel execution of scriptblocks, building indexes for simple and fast search and much more.

So let's start from the very beginning and describe the problem. The main issue is that sometimes it is required to search VMs not only by their names but also by their internal IP addresses or even by name of their disks. Unfortunately i have no idea if there is any API for doing that out of the box. At this point i decided to build this kind of a tool myself. Goals were defined as follows:

- Extract data from Azure as fast as possible, use parallel threads
- Avoid multiple lookups in the array of the VMs. Use some kind of index for this
- Have an ability to add indexes if necessary. Search commands should adopt dynamically
- Use standard, documented REST calls as much as possible
- As far as almost all VMs are V1 use classic REST calls

For those who does not want to read whole article, [code samples are here](https://github.com/eosfor/AzureSearch).

##Extracting data.
In our environment we have more then ten subscriptions and this number going to grow. Lookup procedure should be able to find an object across all existing subscriptions and automagically find new as they appear.  The only way i saw here is to extract all data from Azure and store it locally in some kind of array. I did some tests trying to extract this data using good old PowerShell Azure cmdlets like Get-AzureVM and found performance very slow. It took ages to extract data for only one subscription. So i decided to try to run this cmdlet in parallel using [PSParallel](https://github.com/powercode/PSParallel) module. Unfortunately this was not a good idea. It looks like some of the cmdlets inside of MS Azure module are not ready to run in parallel. I suspect that problem is with Select-AzureSubscription part of it. Parallel execution was failing at some completely unpredictable places with really strange error messages. After some time of experimenting i decided to leave this path and switch to Azure REST API and extract data myself. Drawback of this is that it will be impossible to pass returned objects to other Azure cmdlets. But at least we going to be able find them very fast. Will see how we can overcome this in future.

So I stated with listing cloud services in parallel using [this call](https://msdn.microsoft.com/en-us/library/azure/ee460781.aspx?tduid=(7e6af0dae94cd99be610744b05b54dd4)(256380)(2459594)(XdSn0e3h3.k-wrFcUJSbpSAsQV7.MdbuBQ)()).

<pre class="brush: powershell;">
$sub = Get-AzureSubscription
$h = $sub.SubscriptionId | Invoke-Parallel { Invoke-RestMethod -uri https://management.core.windows.net/$_/services/hostedservices -Method GET -Headers $headers }
</pre>

This works pretty fine and pretty fast. After that i run [get deployment](https://msdn.microsoft.com/en-us/library/azure/ee460804.aspx) to extract VMs data and store it into hashtable with a service name as a key. After this is completed i have a list of all VMs across all of subscriptions available to me.

<pre class="brush: powershell;">
$ht = [hashtable]::Synchronized(@{})
$h.HostedServices.HostedService | skip-null | invoke-parallel {
    #declare function inside of the script block
    function xmlToObject {
        param($o)
        $h = @{}
        $o  | gm -MemberType Property | % {
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
        $d = Invoke-RestMethod -uri '&lt;a href='https://management.core.windows.net/'&gt;https://management.core.windows.net/&lt;/a&gt;$subId/services/hostedservices/$($curr.ServiceName)/deploymentslots/production' -Method GET -Headers $headers
        if ($d) {
            $ri1 = $d.Deployment.RoleInstanceList.RoleInstance | % { xmlToObject $_ }
            $ro1 = $d.Deployment.RoleList.Role | % { xmlToObject $_  }
            $ht[$curr.ServiceName] = $ri1 | % {$c = $_; $r = $c.'RoleName'; $x = $ro1 | where {$_.'RoleName' -eq $r}; $x.remove('RoleName'); [pscustomobject]($_ + $x) }
        }
        else { $ht[$curr.ServiceName] = $null }
    }
    catch { $data = 1 }
}
</pre>

There are couple of points here. First one is that data returned by the call is in somewhat strange format. Information about VM is stored in two different places; in RoleInstanceList sub tree and in RoleList sub tree. In order to simplify further index creation I’m doing conversion of mentioned XML sub trees to hashtables, join them together and produce pscustomobject. After all this new object gets added to a hashtable to the appropriate cloud service. Second thing is that this conversion being done by xmlToObject function, which in turn should be added to each “thread” explicitly. That is why the scriptblock to run contains definition of the xmlToObject function along with REST method invocation.

In general this scriptblock extracts all deployments from the Cloud Service, extracts machine details from the result, makes objects for each machine and add this object to a hashtable under a Cloud Service Key. As a result we have list of VMs indexed by Cloud Service Name.

##Building indexes
At this point we have all the data we need. But if suddenly we need to find something there it may take too much time, especially if we need to do lookup many times. The simplest way to address this is to build additional indexes. For that we can use powershell hashtables. That is pretty easy:

<pre class="brush: powershell;">
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
</pre>

What we need to do here is to run trough the array of VMs, extract a value of the property we are interested in and use it as a key in a hashtable. Value for this key is the object itself. For instance the simplest index would be an index by private IP address of the VMs. In this example we run through objects in the original hashtable, and for each of them extract IP address if the machine and create record in a new hashtable with this IP as a key and current object as the value.

<pre class="brush: powershell;">
$ht.Keys | % {
    $ht[$_] | % {
        if ($_.ipaddress) {$ipAddrIndex[$_.ipaddress] = $ipAddrIndex[$_.ipaddress] + (, $_)}
        else {$ipAddrIndex['noip'] = $ipAddrIndex['noip'] + (, $_)}
    }
}

</pre>`
At this point we have all our data in a hashtable, and couple of indexes. In my case they are by IP address, by storage account and by vhd file names of the VMs. Using them we can search for a VM by using just a semantics of the hashtable like below. It returns all the VMs across all subscriptions having private IP address of 10.10.10.10

<pre class="brush: powershell;">
$ipAddrIndex['10.10.10.10']
</pre>

Next time we going to look at how to build a cmdlet that dynamically builds its parameters set based on names of indexes to do lookups on those indexes.
