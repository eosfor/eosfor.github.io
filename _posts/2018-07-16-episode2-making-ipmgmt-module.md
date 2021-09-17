---
layout: post
title:  '01. Episode 2: Making ipmgmt module'
categories:  ['PowerShell', 'Azure']
tags:  ['PowerShell', 'Azure']
date:  2018-07-16 03:07:56 +03:00
beginps: '<div class="syn_wrapper"><pre class="brush: powershell;">'
endps: '</pre></div>'
---

Hello colleagues, we continue building our Enterprise-focused Azure automation module. And now lets took at the task from a different angle. So far we managed to create name generation components, and what we need to do now? Well, if we need to automate VM provisioning we need one more thing - networks. We need to be able to automate network creation, meaning, we need to automate IP ranges calculations and lookups for free IP ranges in our estate. So lets focus on this

# Goals

In this episode we will create another module which we will later use to automate network provisioning. And things we want to cover here are:

- Given a VNET range break it down to number of subnets based on a subnet size
- Given a list of IP ranges in use and  number of IP addresses needed find out a free slot for a VNET

## Rationale

So far I've seen two different approaches for hybrid networking. First approach was to create a separate network for an application or a set of applications, and give some control away to an application team, so they can put anything they like to that network, with respect to security requirements, of course.

Second approach was more strict. This one is bout creating number of big security zones, under control of a security or a networking team. And app team should get their applications into subnets under these big zones.

In both cases there are two main issues while provisioning VNETs in the cloud. First is we need to find a proper slot and second - we need to break down a VNET to subnets. This has more value if an organization follows the first approach. In that case it needs to create more networks, and this should be done by request.

## Implementation

In or implementation we will first focus on breaking a VNET to subnets, this is more common task to perform. It is useful not only for Azure provisioning. And during our work on this, in terms of using PowerShell  we will look at the following things:

- Using PowerShell [PackageManagement](https://docs.microsoft.com/en-us/powershell/module/packagemanagement/?view=powershell-6) module
- Using .NET classes in PowerShell
- Some interesting syntax of functions
- A bit of Pester tests

### Attaching Ipnetwork2

So lets start. In our work we don't want to reinvent any wheels, so let just use something that already exists. Here we are going to use [Ipnetwork2](https://github.com/lduchosal/ipnetwork) library. For that we use [PackageManagement](https://docs.microsoft.com/en-us/powershell/module/packagemanagement/?view=powershell-6) module like this. First check if it is there, and then install it

{{ page.beginps }}
Find-Package ipnetwork2
Install-Package ipnetwork2
{{ page.endps }}

This is it. Next, we want to redistribute our module with this package, as it is going to be easer for us. So we need to find a proper dll to pick up. Below command will return the place, where the package is stored, and under ```lib\netstandard1.3``` folder we will find the netstandard-based System.Net.IPNetwork.dll which is what we need. We will use it and distribute it with our module, so we copy it to the /bin folder inside of our module.

{{ page.beginps }}
(get-Package ipnetwork2).source
{{ page.endps }}

Great, so what is next? Next we start using it by loading it into our module. For that we set the following parameter in our ipmgmt.psd1 file, this will load this assembly into the powerShell session.

{{ page.beginps }}
RequiredAssemblies = @('.\bin\System.Net.IPNetwork.dll')
{{ page.endps }}

Now, as usual we add a file which will contain our networking code, and dot-source it

{{ page.beginps }}
Get-ChildItem $PSScriptRoot\*.ps1 | ForEach-Object {
    write-verbose "dot sourcing $($_.FullName)"
    . $_.FullName
}
{{ page.endps }}

And now we are ready to make our first cmdlet in this module.

### Implementing VLSM breakdown

So, our new cmdlet should take an IP range in a form of CIDR notation, i.e. 10.10.5.0/24, and a set of subnets. Commonly companies may have different layout for a network, so what we will do here is we give a possibility to provide a layout and subnet names, like this:

{{ page.beginps }}
$subnets =
    @{type = "GTWSUBNET"; size = 30},
    @{type = "DMZSUBNET"; size = 62},
    @{type = "BKESUBNET"; size = 30},
    @{type = "APPSUBNET"; size = 62}
{{ page.endps }}

We don't want our users to think about calculating masks or something. This should be done by a computer. We don't want them to check if something will be left unused in this network, we want our command to do all of the stuff. All we want from our users is to provide subnets in a proper form. We want to have subnet names and sizes, meaning number of addresses they want to use, in each of these subnets. This is it. In this case user experience will be something like this:

{{ page.beginps }}
Get-VLSMBreakdown -Network 10.10.5.0/24 -SubnetSize $subnets
{{ page.endps }}

Looks easy, right? Yeah, but implementation has some interesting point so look at. Here it is

{{ page.beginps }}
function Get-VLSMBreakdown {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        [System.Net.IPNetwork]$Network,

        [Parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        [array]$SubnetSize
    )

    function processRecord($net, $cidr, $type) {
        try {
            #try breaking down a network block by CIDR in a form of length
            $subnets = @([System.Net.IPNetwork]::Subnet($net, $cidr))

            #if success push the first resulting subnet to output stack and mark them with associated type
            $outStack.Push(($subnets[0] | add-member -MemberType NoteProperty -Name type -Value $type -PassThru -Force))

            #if there are more subnets generated by "subnet" operation
            #returning the rest subnets to the Ip blocks queue fur further processing
            if ($subnets.count -gt 1) {
                for ($i = 1; $i -le $subnets.count - 1; $i++) {
                    $vlsmStack.Enqueue($subnets[$i])
                }
            }
            # return true if "subnetting was successfull"
            $true
        }
        catch {
            # put the network back to the IP blocks queue
            $vlsmStack.Enqueue($net)
            # return true if "subnetting failed"
            $false
        }
    }


    if (($SubnetSize | ForEach-Object {[PSCustomObject]$_} | Measure-Object -Property size -Sum).Sum -lt $Network.Usable) {
        #queue of masks we what to use to break our network
        $vlsmMasks = [System.Collections.Queue]::new()

        #calculating mask length based on subnet sizes we get as an input
        #as an output we get list of objects (type; length) sorted by length
        #list is stored in $vlsmMasks variable
        $SubnetSize | ForEach-Object {
            $length = 32 - [math]::Ceiling([math]::Log($_.size + 2, 2))
            [PSCustomObject]@{
                type   = $_.type;
                length = $length
            }
        } | Sort-Object -Property length | ForEach-Object {$vlsmMasks.Enqueue($_)}

        #queue of networks to break down
        $vlsmStack = [System.Collections.Queue]::new()
        #stack of subnets we going to break the network to
        $outStack = [System.Collections.Stack]::new()
        #put the very first network to the queue
        $vlsmStack.Enqueue($Network)

        #at this point we got two queues:
        #masks queue and networks queue
        #masks hosls lengths of all subnets we need and netwoks contains our VNET range (network block)

        do {
            $failureCount = 0
            #pick a msk form a queue
            $v = $vlsmMasks.Dequeue()
            try {
                #pick a network block form a queue
                $current = $vlsmStack.Dequeue()
            }
            catch {
                write-verbose -Message "No networks in the queue to process"
            }

            #processing the block against the mask and assosiated type
            #if no success - return mask back to tail of a queue and increase failue count
            if (! (processRecord $current $v.length $v.type) ) { $vlsmMasks.Enqueue($v); $failureCount++}

            write-verbose ("VLSM MASKS`: " + $vlsmMasks.Count)
            write-verbose ("FAILURES`: " + $failureCount)

            if ($vlsmMasks.Count -eq $failureCount) {break} # break when no records processed during a loop cycle
        } while ($vlsmMasks.Count -ne 0) #leave when we have masks queue emty

        write-verbose ("OUT STACK`: " + $outStack.Count)
        write-verbose ("SubnetSuize`: " + $SubnetSize.count)
        if ($outStack.count -lt $SubnetSize.count) {write-warning -message "subnetting failed"}

        #in case we have more subnets than requested store them
        $reserved = @([System.Net.IPNetwork]::Supernet($vlsmStack.ToArray()))
        $outStack
        #and mark them as 'reserved'
        $reserved | add-member -MemberType NoteProperty -Name type -Value "reserved" -PassThru -Force
    }
}
{{ page.endps }}

Pretty big, right? Well, if I were to remove all comments, it would be twice shorter, but twice more spooky :). So lets quickly go through it, just on a high level. 

We take an IP range in a form of a string and immediately parse it into a ```[System.Net.IPNetwork]``` type, which comes from a library we load with our module. The second parameter is a list of subnets in a for of their name (I call it "type" for a security zone type). 

So first thing we do is we take these numbers and calculate subnet length, in bits, which will be able to accommodate all requested IPs. At the end of this we receive a list of ```type; length``` objects, and then we sort them by length and put into a queue. Lets call this "masks" queue.

Next thing we do is we put or network we got as a parameter to another queue, which hosts our networks.

Now, we do simply following:

- Pick a mask from a "masks" queue
- Pick a network from a networks queue
- Try breaking this network by the mask
  - If it "breaks" then remove both from their queues, and put to the output object
  - If it does not - put the network to the back of their queue and start over

Well, of course this is the high level, there are few extra steps, but this is the general idea. So here is what we get as a result:

![vlsm results](/images/posts/2018-07-16-episode2-making-ipmgmt-module/vlsmResults.png)

Which is nice and we can easily use this in our "automations".

### Some pester tests

Well, as you probably know it is good idea to test your code, in some automatic manner. When using PowerShell it is pretty easy to do with [Pester](https://github.com/pester/Pester). Our [Plaster](https://github.com/PowerShell/Plaster) template, as you might recall from the earlier episodes, creates the test file for us, so we can add the following code to it.

Closer to the top we add this, so Pester will load our module

{{ page.beginps }}
$modulePath = "$PSScriptRoot\.."

Import-Module $modulePath -Verbose
{{ page.endps }}

and then we add a simple test

{{ page.beginps }}
Describe 'Simple test' {
    It 'Passes a simple test' {
        $subnets =
        @{type = "GTWSUBNET"; size = 30},
        @{type = "DMZSUBNET"; size = 62},
        @{type = "EDGSUBNET"; size = 30},
        @{type = "APPSUBNET"; size = 62},
        @{type = "CRESUBNET"; size = 62}

        Get-VLSMBreakdown -Network 10.10.5.0/24 -SubnetSize $subnets
    }
}
{{ page.endps }}

This is how it looks at the end:

![pester run](/images/posts/2018-07-16-episode2-making-ipmgmt-module/pester.png)