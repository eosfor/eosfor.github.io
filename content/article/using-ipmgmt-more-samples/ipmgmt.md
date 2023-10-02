# How to automate IP ranges calculations in Azure using PowerShell

> The notebook does work on Linux and [![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/eosfor/scripting-notes/HEAD)

## Scenario

Assume we got the IP rage of `10.172.0.0/16` from the network team for planned Azure Landing Zones. We want to automate this by making a tool, which will automatically calculate IP ranges for us given some high level and simple to understand details about the future networks.

This notebook shows how to do it using the [ipmgmt](https://github.com/eosfor/ipmgmt) module. So let us go and install it first


```C#
Install-Module ipmgmt -Scope CurrentUser
```

And import it


```C#
Import-Module ipmgmt
```

The module contains only two cmdlets. The `Get-VLSMBreakdown` breaks down a range into smaller ones, meaning we can use it to break a range into VNETs and then each VNET into subnets. The `Get-IPRanges` cmdlet given the list of ranges in-use and the "root" range tries to find a free slot of the specified size. It can be used to avoid losing IP space


```C#
Get-Command -Module ipmgmt
```

    
    [32;1mCommandType     Name                                               Version    Source[0m
    [32;1m-----------     ----                                               -------    ------[0m
    Function        Get-IPRanges                                       0.1.5      ipmgmt
    Function        Get-VLSMBreakdown                                  0.1.5      ipmgmt
    


Let us look at how we can break down our big "root" IP range into smaller ones. For that, we just need to prepare a list of smaller sub-ranges in the form of PowerShell hashtables, like so `@{type = "VNET-HUB"; size = (256-2)}`. Here we say that the name of the range is `VNET-HUB` and the size is simply `256-2`, which is the maximum number of IPs in the `/24` subnet minus 2, the first and the last.

If we need more than one, we just make an array of these hashtables


```C#
$subnets = @{type = "VNET-HUB"; size = (256-2)},
           @{type = "VNET-A"; size = (256-2)}
```

Now everything is ready and we can try to break the "root" network


```C#
Get-VLSMBreakdown -Network 10.172.0.0/16 -SubnetSize $subnets | ft type, network, netmask, *usable, cidr -AutoSize
```

    
    [32;1mtype     Network      Netmask       FirstUsable  LastUsable     Usable Cidr[0m
    [32;1m----     -------      -------       -----------  ----------     ------ ----[0m
    VNET-A   10.172.1.0   255.255.255.0 10.172.1.1   10.172.1.254      254   24
    VNET-HUB 10.172.0.0   255.255.255.0 10.172.0.1   10.172.0.254      254   24
    reserved 10.172.128.0 255.255.128.0 10.172.128.1 10.172.255.254  32766   17
    reserved 10.172.64.0  255.255.192.0 10.172.64.1  10.172.127.254  16382   18
    reserved 10.172.32.0  255.255.224.0 10.172.32.1  10.172.63.254    8190   19
    reserved 10.172.16.0  255.255.240.0 10.172.16.1  10.172.31.254    4094   20
    reserved 10.172.8.0   255.255.248.0 10.172.8.1   10.172.15.254    2046   21
    reserved 10.172.4.0   255.255.252.0 10.172.4.1   10.172.7.254     1022   22
    reserved 10.172.2.0   255.255.254.0 10.172.2.1   10.172.3.254      510   23
    


Here we got two ranges named `VNET-A` and `VNET-HUB`. However, by doing so we made a few unused slots in the `root` range. They are marked as `reserved`, just for our convenience. It shows what happens to the range when we break it down. The smaller sub-ranges you make, the more of such unused ranges you get in the end.

Another way of doing this is by using CIDR notation, like this


```C#
$subnets = @{type = "GTWSUBNET"; cidr = 27},
@{type = "DMZSUBNET"; cidr = 26},
@{type = "EDGSUBNET"; cidr = 27},
@{type = "APPSUBNET"; cidr = 26},
@{type = "CRESUBNET"; cidr = 26}

Get-VLSMBreakdown -Network 10.10.5.0/24 -SubnetSizeCidr $subnets | ft -AutoSize
```

    
    [32;1mtype      Network     AddressFamily Netmask         Broadcast   FirstUsable LastUsable  Usable Tota[0m
    [32;1m                                                                                                  l[0m
    [32;1m----      -------     ------------- -------         ---------   ----------- ----------  ------ ----[0m
    EDGSUBNET 10.10.5.224  InterNetwork 255.255.255.224 10.10.5.255 10.10.5.225 10.10.5.254     30   32
    GTWSUBNET 10.10.5.192  InterNetwork 255.255.255.224 10.10.5.223 10.10.5.193 10.10.5.222     30   32
    CRESUBNET 10.10.5.128  InterNetwork 255.255.255.192 10.10.5.191 10.10.5.129 10.10.5.190     62   64
    APPSUBNET 10.10.5.64   InterNetwork 255.255.255.192 10.10.5.127 10.10.5.65  10.10.5.126     62   64
    DMZSUBNET 10.10.5.0    InterNetwork 255.255.255.192 10.10.5.63  10.10.5.1   10.10.5.62      62   64
    


Ok, let's try to use what we've got. For that, we need to authenticate to Azure. When running locally you can just do


```C#
Login-AzAccount
```

In Binder however, it needs to be a bit different, like this


```C#
Connect-AzAccount -UseDeviceAuthentication
```

Once authenticated, we can create networks, for example, like this. Here we first filter out the `reserved` ones for simplicity.


```C#
$vnets = Get-VLSMBreakdown -Network 10.172.0.0/16 -SubnetSize $subnets | ? type -ne 'reserved'

$vnets | % {
    New-AzVirtualNetwork -Name  $_.type -ResourceGroupName 'vnet-test' `
                         -Location 'eastus2' -AddressPrefix "$($_.Network)/$($_.cidr)" | select name, AddressSpace, ResourceGroupName, Location
}
```

Ok, assume, at some point, we need to add a few more networks. And at the same time e may want to reuse one of those `reserved` slots, if it matches the size. This is what `Get-IPRanges` does. It takes a lit of IP ranges "in-use" and returns slots that can fit the range in question. For example, in our case, we have a "base range" of `10.10.0.0/16` and two ranges in-use `10.10.5.0/24`, `10.10.7.0/24`. We are looking for a range of the size `/22`. So the cmdlet recommends us to use the `10.172.4.0/22`, which is one of the `reserved` ranges from the previous example


```C#
Get-IPRanges -Networks "10.172.1.0/24", "10.172.0.0/24" -CIDR 22 -BaseNet "10.172.0.0/16" | ft -AutoSize
```

    
    [32;1mIsFree Network    AddressFamily Netmask       Broadcast    FirstUsable LastUsable   Usable Total Ci[0m
    [32;1m                                                                                                 dr[0m
    [32;1m------ -------    ------------- -------       ---------    ----------- ----------   ------ ----- --[0m
     False 10.172.0.0  InterNetwork 255.255.255.0 10.172.0.255 10.172.0.1  10.172.0.254    254   256 24
     False 10.172.1.0  InterNetwork 255.255.255.0 10.172.1.255 10.172.1.1  10.172.1.254    254   256 24
      True 10.172.4.0  InterNetwork 255.255.252.0 10.172.7.255 10.172.4.1  10.172.7.254   1022  1024 22
    


What if we need to find more than just one range at a time? Need not worry. We can do it with this simple script. And we are using Azure as a source of truth, as we can always query it for the real IP ranges, which are in use.

So what we need to do is relatively simple:

- make a list of sizes we want to create and put it into a variable - `$cidrRange`
- pull the ranges from Azure. We assume they are in use by someone - `$existingRanges`
- cast whatever we pulled from azure the `System.Net.IPNetwork` for correctness. This type is used inside the `ipmgmt` module to store information about networks and do all the calculations, comparisons, etc.
- now we run through the list of sizes, for each of them ask `Get-IPRanges` to find a proper slot, and accumulate the results

Now we just need to mark the new ranges as `free`, to see what we've got. For that,  we compare what we have in Azure to what we just calculated and mark the difference accordingly


```C#
$cidrRange = 25,25,24,24,24,24,23,25,26,26 | sort
$existingRanges = (Get-AzVirtualNetwork -ResourceGroupName vnet-test | 
    select name, @{l = "AddressSpace"; e = { $_.AddressSpace.AddressPrefixes }}, ResourceGroupName, Location |
    select -expand AddressSpace)
$existingNetworks = $existingRanges | % {[System.Net.IPNetwork]$_}
$nets = $existingRanges

$ret = @()

$cidrRange | % {
    $ret = Get-IPRanges -Networks $nets -CIDR $_ -BaseNet "10.172.0.0/16"
    $nets = ($ret | select @{l="range"; e = {"$($_.network)/$($_.cidr)"}}).range
}

$ret | % {
    if ( -not ($_ -in $existingNetworks)) {$_.IsFree = $true}
}

$ret | ft -AutoSize
```

    
    [32;1mIsFree Network      AddressFamily Netmask         Broadcast    FirstUsable  LastUsable   Usable Tot[0m
    [32;1m                                                                                                 al[0m
    [32;1m------ -------      ------------- -------         ---------    -----------  ----------   ------ ---[0m
     False 10.172.0.0    InterNetwork 255.255.255.0   10.172.0.255 10.172.0.1   10.172.0.254    254 256
     False 10.172.1.0    InterNetwork 255.255.255.0   10.172.1.255 10.172.1.1   10.172.1.254    254 256
      True 10.172.2.0    InterNetwork 255.255.254.0   10.172.3.255 10.172.2.1   10.172.3.254    510 512
      True 10.172.4.0    InterNetwork 255.255.255.0   10.172.4.255 10.172.4.1   10.172.4.254    254 256
      True 10.172.5.0    InterNetwork 255.255.255.0   10.172.5.255 10.172.5.1   10.172.5.254    254 256
      True 10.172.6.0    InterNetwork 255.255.255.0   10.172.6.255 10.172.6.1   10.172.6.254    254 256
      True 10.172.7.0    InterNetwork 255.255.255.0   10.172.7.255 10.172.7.1   10.172.7.254    254 256
      True 10.172.8.0    InterNetwork 255.255.255.128 10.172.8.127 10.172.8.1   10.172.8.126    126 128
      True 10.172.8.128  InterNetwork 255.255.255.128 10.172.8.255 10.172.8.129 10.172.8.254    126 128
      True 10.172.9.0    InterNetwork 255.255.255.128 10.172.9.127 10.172.9.1   10.172.9.126    126 128
      True 10.172.9.128  InterNetwork 255.255.255.192 10.172.9.191 10.172.9.129 10.172.9.190     62  64
      True 10.172.9.192  InterNetwork 255.255.255.192 10.172.9.255 10.172.9.193 10.172.9.254     62  64
    


And this gives us all the necessary information to add a few more networks.
