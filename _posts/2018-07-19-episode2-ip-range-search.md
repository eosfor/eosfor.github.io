---
layout: post
title:  '02. Episode 2: What if we need to find a free IP Range?'
categories:  ['PowerShell']
tags:  ['PowerShell']
date:  2018-07-19 05:57:27 +03:00
beginps: '<div class="syn_wrapper"><pre class="brush: powershell;">'
endps: '</pre></div>'
---

Hello colleagues, we continue building our Enterprise-focused PowerShell module for Azure. And this time we will extend functionality of our IP management module.

# Some considerations

As you might recall we making a module, which will have the following functionality.

- Given a VNET range break it down to number of subnets based on a subnet size
- Given a list of IP ranges in use and  number of IP addresses needed find out a free slot for a VNET

Typically, when we need to provision an VM-based environment in Azure we need to have a network, to put our VMs into. To create such a network automatically, we need to know few things. First - we need to know IP Range for a VNET. Next is the subnet layout, like which subnets we want to create, what will be their IP ranges. Most commonly, however, we know which ranges are in use - this information can be extracted from a CMDB or directly form Azure. And we can also estimate sizes for our subnets. Having the list of sizes we now able to calculate a VNET layout using cmdlet we developed [last time](https://eosfor.github.io/2018/episode2-making-ipmgmt-module/). Now we need to develop something, that can find a proper slot for us, among all the VNETs we may already have.

## Assumptions

We will assume here, that we have some kind of an [IPAM](https://docs.microsoft.com/en-us/windows-server/networking/technologies/ipam/ipam-top) system in our infrastructure, and either can extract this date from it, or, it may be stored in CMDB. I'd recommend however, to build some automated mechanism, which will update CMDB or IPAM with new networks in Azure automatically, upon creation. We can, probably, look at this later and build a prototype. So, by now we assume that this data is available for us.

Typically, when an Enterprise builds a hybrid networking, few big chunks of IP address space are dedicated for regions in a cloud. For example if we have some workloads to run in the US, some in the EU, and some in Australia - we may have three different big IP ranges of, say, size "/16". In our terminology we going to call these - "base networks" or "base ranges". Each of these ranges will have some VNETs inside, so ur goal is to search through IP ranges under a base network, to find a free slot of the size we need.

# What it looks like

This is how it works, and now lets look at how we did it.

![how search works](/images/posts/2018-07-19-episode2-ip-range-search/searchDemo/searchDemo.gif)

# Implementation

Well, frankly speaking implementation looks pretty, mm, complex. But in fact it is not. Well, I'd say, I'd like someone with really strong programming background to look at this to point out what is wrong with this. But, well, now it more or less works and we used it a lot in Production and we haven't seen any problems. So, lets look at this closer and put scary things first:

{{ page.beginps }}
function Get-IPRanges {
    [cmdletbinding()]
    param(
        [Parameter(Mandatory = $true, ParameterSetName = "ByBaseNet")]
        $Networks,
        [Parameter(Mandatory = $true, ParameterSetName = "ByBaseNet")]
        [System.Net.IPNetwork] $BaseNet,
        [Parameter(Mandatory = $true, ParameterSetName = "ByBaseNet")]
        [ValidateRange(8, 30)]
        [int] $CIDR
    )
    function NextSubnet ($network, $newCIDR ) {
        $net = [System.Net.IPNetwork]::Parse($network)
        $netmask = [System.Net.IPNetwork]::ToNetmask($newCIDR, $net.AddressFamily)

        [bigint] $nextAddr = [System.Net.IPNetwork]::ToBigInteger($net.Broadcast) + 1
        [bigint] $mask = [System.Net.IPNetwork]::ToBigInteger($netmask)
        [bigint] $masked = $nextAddr -band $mask

        if ($masked -eq $nextAddr) {
            Write-Verbose "Masked"
            $next = $masked
        }
        else {
            Write-Verbose "Magic!"
            $next = $masked + [bigint]::Pow(2, 32 - $newCIDR)
        }

        $nextNetwork = [System.Net.IPNetwork]::ToIPAddress($next, [System.Net.Sockets.AddressFamily]::InterNetwork)
        $nextSubnet = [System.Net.IPNetwork]::Parse($nextNetwork, $netmask)
        $nextSubnet
    }
    function getIPRanges {
        for ($i = 0; $i -le ($outNets.Count - 1); $i++ ) {
            $n = $outNets[$i]
            $ns = NextSubnet $n $CIDR
            Write-Verbose "subnet is`: $n; next subnet is $ns"

            # test if 'next subnet' is a part of a base range
            if (-not [System.Net.IPNetwork]::Contains($BaseNet, $ns)) {
                Write-Verbose "$ns is not in a $BaseNet"
                # skip the network if it is not in a base range
                continue;
            }
            else {Write-Verbose "$ns is in a $BaseNet"}

            # test if 'next subnet' overlaps any of the existing below it in a sorted list
            $k = $i; $isoverlap = $false
            do {
                if ([System.Net.IPNetwork]::Overlap($outNets[$k], $ns)) {
                    Write-Verbose "$ns overlaps $($outNets[$k])"
                    # break this loop if there is an overlap
                    $isoverlap = $true
                    break
                }
                $k++
            } while ($k -lt ($outNets.Count))

            # when there were no overlaps -> output
            if (! $isoverlap) {
                Write-Verbose "$ns did not overlap"
                $ns
            }
        }
    }

    $calcNets = {
        # put all networks we've found to the out collection, sort them
        # for some reason sort cmdlet does not work well, so use Lists and internal comparer.
        $outNets = [System.Collections.Generic.List[System.Net.IPNetwork]]::new()
        $Networks | ForEach-Object {
            $outNets.add($_)
        }
        $outNets.Sort()

        # create a collection of unused nets
        $freeNets = [System.Collections.Generic.List[System.Net.IPNetwork]]::new()

        # mark all existing as 'used'
        $outNets | Add-Member -MemberType NoteProperty -Name IsFree -Value $false -Force

        # search for free ranges in between the items of the 'used' nets list and after the last one
        $n = getIPRanges
        Write-Verbose "ip ranges we've got`: $n"

        # if there is something - add them to 'unused' list
        # mark them as 'unused'
        if ($n.count -gt 0) {
            $n | ForEach-Object {
                $net = $_ | Add-Member -MemberType NoteProperty -Name IsFree -Value $true -Force -PassThru;
                $freeNets.add($net)
            }
        }

        # TODO: search networks in front of the provided set of used networks
        $firstFree = [System.Net.IPNetwork]::ToBigInteger($BaseNet.FirstUsable)
        $lastFree = [System.Net.IPNetwork]::ToBigInteger($outNets[0].FirstUsable)
        $diff = $lastFree - $firstFree
        if ($diff -gt 0) {
            Write-Verbose "there are addresses in front of occupied blocks, number is`: $diff"
            $frontNets = [System.Collections.Generic.List[System.Net.IPNetwork]]::new()
            $firstNetToTest = "$($BaseNet.FirstUsable.ToString())/$CIDR"
            $netToTest = [System.Net.IPNetwork]::Parse($firstNetToTest)


            while (-not [System.Net.IPNetwork]::Overlap($outNets[0], $netToTest)) {
                $netToTest | Add-Member -MemberType NoteProperty -Name IsFree -Value $true -Force
                $frontNets.Add($netToTest)
                $netToTest = NextSubnet $netToTest $CIDR
            }

            Write-Verbose "$frontNets"
            $freeNets.AddRange( $frontNets )
            $freeNets.Sort()
        }
        # join used and unused together to output them properly
        $outNets.AddRange( $freeNets )
        $outNets.Sort()

        # sometimes two consequent subnets may end up with the same next free subnet. selecting unique values to avoid confusion
        $outNets | Select-Object -Unique

    }

    & $calcNets
}
{{ page.endps }}

## Parameters

Based on our assumptions we take few parameters. First of all it is the base network or the base range, we want to use. In general this defines which global range will be used to pick a subnet from. Next - the list of ranges which are already in use. And the last one is the CIDR or length of the new range we are looking for. The commandlet will look for a first free subnet of the specified size inside of the specified base range keeping in mind all used blocks.

## Some implementation details

Well, in general we do the following, on the high level:

- All network ranges passed as "used" converted to ```[System.Net.IPNetwork]```, put to a ```List```, sorted and marked as "used". Add them to the output of the cmdlet
- Search the list for free slots of the required size
  - For each of the networks in the least calculate the "Next Range" of the required size
  - Test "Next Range" to be a part of the base network
  - Test "Next Range" not to overlap with other existing networks
  - If both chars are OK, put the range to the output
- Pick the output from the search procedure, mark all the things found as "unused" and put them to the output of the cmdlet
- Look if there is a free range in front of the set of all "used" networks. If it is, mark it accordingly and put to the output

So the "search" procedure is defined as a function in the scope of the cmdlet. Meaning that we declare this function inside of the body of the cmdlet (```getIPRanges```). In addition to this we have another such function - ```NextSubnet```, which basically represents operation of calculating the "Next Range".

Also, for troubleshooting purposes I added ```-Verbose``` parameter with excessive logging, so you can see what is happening live.

# Usage scenarios

Well, the first scenario we used this for was the following:

- Pick the request for network creation, which contained the Azure Region name to put the network to, and the VNET breakdown in a form of just set of sizes, like for example (10,10,10,10). Our default layout was static and consisted of four subnets.
- Automatically determine the base range from the CMDB, based on the Region provided
- Calculate a VNET size by just adding together VNET sized, requested by a user
- Search for a free slot based on the VNET size, base network and the list of networks in use from the CMDB
- Calculate VLSM breakdown for the free slot we just found
- Fire the ARM template, providing VNET range, Subnet ranges, Names etc.

Well, this, in general, how it worked. So I hope this will also help you with your automations!