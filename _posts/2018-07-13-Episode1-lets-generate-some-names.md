---
layout: post
title:  "07. Episode 1: Let's generate some names"
categories:  ['PowerShell', 'Azure']
tags:  ['PowerShell', 'Azure']
date:  2018-07-13 04:49:29 +03:00
beginps: '<div class="syn_wrapper"><pre class="brush: powershell;">'
endps: '</pre></div>'
---

Ok, this moment is here! Now we can make a cmdlet, that will generate names for us.

# Let's review why we are doing this

First of all let's see why we need this and why we've put all of these efforts. Here is the slide

![name generation slide](/images/posts/2018-07-13-Episode1-lets-generate-some-names/nameGenerationSlide.png)

So far we've developed number of components which provide our replaceable interface to the cloud-based CMDB. This interface consists of some components, which understand what this CMDB is, where it is and how to query this CMDB. In addition to that they include caching, so all CMDB data is cached locally to speedup user experience of our PowerShell module. And on top of all that we got an abstraction, which allows us to quickly reconfigure our module with another CMDB with minimal code change.

All of this allows us to start generating names according to the corporate standard using organization structure data, stored in CMDB. Which in turn allows, us to be automatically compliant to corporate standards!

In addition to that, during development we looked at following techniques:

- PowerShell modules
- Plaster
- PowerShell classes
- Process of discovering internals of PowerShell cmdlets and .NET types
- Usage Azure Storage and Azure tables from PowerShell
- Hashtables in PowerShell
- PowerShell-based Singleton and it's benefits
- Parameter value completion options

And one of the biggest things, I think, is that we are going through the process of building this module step by step, so everyone can see how easy it is to make automation on PowerShell.

# Making our name generation work

So, let's continue. First thing we need to do is to define naming convention. We assume this has been done by our Architecture Team and provided to us, so here is how it looks like:

## Architecture guidelines

General resource naming should follow the scheme:

> \<BUCODE\>-\<DEPTCODE\>-\<ENVCODE\>-\<AZREGION\>-\<RESOURCECODE\>-\<SUFFIX\>
>
> Where:
>
> - **BUCODE** is the Business Unit code, taken from CMDB. Identifies the owner of the resource. 3 chars long
> - **DEPTCODE** is the Department Code, taken from CMDB. Identifies the owner of the resource. 3 chars long
> - **ENVCODE** [DEV,UAT,PDN], specifies the Environment, development testing or production. 3 chars long
> - **AZREGION** Azure region code, taken from CMDB. contains Location code, Country code and Number to provide uniqueness. 5 chars total
> - **RESOURCECODE** identifies the type of the resource [here goes the list of resource codes and their meanings ;)]. 3 chars long
> - **SUFFIX** provides some meaning to owner Team and makes the whole name unique. Depends on the length limitations for particular resource > type
>
> In addition to all this there are following limitations/restrictions
>
> - Virtual Machine host name is limited to 15 characters, so the naming scheme for it is: <BUCODE><DEPTCODE><ENVCODE><SUFFIX>
> - Azure Storage accounts has limitations on the structure of their name as well as on its length - 24 chars.
>
> All other resources should follow general naming approach.

## Implementation

This is something we can now follow, as we are able to talk to CMDB and extract all necessary components. And here is what we need to do

```powershell
function New-AzureResourceName {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory=$true)]
        [ArgumentCompleter([CMDBBUCompleter])]
        [string]$BUName,

        [Parameter(Mandatory=$true)]
        [ArgumentCompleter([CMDBDEPTCompleter])]
        [string]$DEPTName,

        [Parameter(Mandatory=$true)]
        [ArgumentCompleter([AZRegionCompleter])]
        [string]$AZRegion,

        [Parameter(Mandatory=$true)]
        [ValidateSet("DEV","UAT","PDN")] # we want to enfoce these
        [string]$EnvCode,

        [Parameter(Mandatory=$true)]
        [ArgumentCompletions("AVM", "STR")] # there are special cases here. we need to process them separately
        [string]$ResourceCode,

        [Parameter(Mandatory=$true)]
        [string]$Suffix
    )


    end {
        $name = ""
        $vmNameErrorMsg = "VM Name length should be less than 15 characters"
        $strAcctErrorMessage = "Resource Name length should be less than 24 characters"

        switch ($ResourceCode) {
            "AVM" { $name = ($BUName + $DEPTName + $EnvCode + $Suffix).ToUpper(); if ($name.Length -gt 15) {Write-Error $vmNameErrorMsg -EA Stop } break;}
            "STR" { $name = ($BUName + $DEPTName + $EnvCode + $AZRegion + $ResourceCode + $Suffix).ToLower(); if ($name.Length -gt 24) {Write-Error $strAcctErrorMessage -EA Stop } break}
            Default {$name =(($BUName, $DEPTName, $EnvCode, $AZRegion, $ResourceCode, $Suffix) -join '-').ToUpper()}
        }
        $name
    }
}
```

This one is pretty big, but we have already seen parts of it. First of all we define all parameters for our cmdlet as mandatory. We want to force our users to provide them all. Next thing we did is, we defined completion for almost all parameters. Here we trying to make sure that there will be no room for human error, and all names will be forced to follow the standard. Everything else is more or less straightforward. In order to apply limitations to resource names we defined some resource types, and if we hit them we do some checks and trigger terminating errors in case checks are failed.

# Changes we did to completion objects

The biggest change we had to do was to the completers part. I completely changed the way I use completers, from using functions to using classes. I've completely removed ```completers.``` file and moved all completers into a separate file with all classes in it. At the very top, to save some time on typing and prettify the code I add this

```powershell
using namespace System.Management.Automation
using namespace System.Management.Automation.Language
using namespace System.Collections
using namespace System.Collections.Generic
```

This basically specifies namespaces to look for classes. At this point we don't need to specify the the long name for a class. You can find a bit more about this syntax [here](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_using?view=powershell-6). Let's look at some examples

```powershell
class CMDBDEPTCompleter : IArgumentCompleter
{
    [IEnumerable[CompletionResult]] CompleteArgument(
        [string] $CommandName,
        [string] $parameterName,
        [string] $wordToComplete,
        [CommandAst] $commandAst,
        [IDictionary] $fakeBoundParameters)
    {
        $resultList = [List[CompletionResult]]::new()

        $buParamValue = $fakeBoundParameters.BUName
        $BUs = (& $Global:cmdbFunc).Where({$_.PartitionKey -EQ "DEPT"}). # <- where method
                                    Where({$_.BU -EQ $buParamValue}).
                                    Where({$_.Name -like "*$wordToComplete*"})

        foreach ($b in $BUs) {
            $resultList.Add([CompletionResult]::new($b.code, $b.name, "ParameterValue", 'DEPT'))
        }

        return $resultList
    }
}
```

This class, as you can see from it's name, is to auto-complete Departments parameter. Well, I should mention that you can use the same class to any BU/DEPT parameter pair on any cmdlet in our module, so we save a lot of typing. Following [```IArgumentCompleter```](https://docs.microsoft.com/en-us/dotnet/api/system.management.automation.iargumentcompleter?view=powershellsdk-1.1.0) interface definition, we must implement ```CompleteArgument``` method, and we do it. There are few interesting syntactic things, however, to pay attention to. 

First of all I'm trying to avoid using pipelines in classes. Well, it is ok to use them, but pipelines are "a bit" slower, compared to regular loops, so to make our auto-completion a bit faster I decided to rewrite it with no pipelines. However for the ```CMDBBUCompleter``` I'll do this later :).

Second thing is, using [```.where```](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_arrays?view=powershell-6#where) method. This is another option to speed filtering a bit up. Basically what it does is it runs the scriptblock passed as a parameter against each object in the array. And returns the object if true, so you can "chain" these calls for more advanced filtering.

Third is this strange ```Where({$_.Name -like "*$wordToComplete*"})``` piece. Actually we saw this previously, and what it does is, each time you hit tab the ```CompleteArgument``` function gets called and ```$wordToComplete``` variable is filled with the part of the name you typed on the command line, so you can partially type the value of the argument, and this filter will find the exact value by what you typed.

As you can see also we still use our ```$Global:cmdbFunc``` which makes it possible to switch CMDB when needed.

Well, and as the last step we fill in the list with values found and return it. Just to be clear the syntax ```[List[CompletionResult]]``` does the same stuff as ```List<CompletionResult>```, meaning this is PowerShell way to declare a generic collection of a specific type. So we declare and instantiate a List of CompletionResult and then fill it in and return to the caller.

So, at this point what we've got is a name generating function, which uses CMDB, makes sure you've specified all your parameter values correctly (trough completion), checks against restrictions provided and returns a name.

Next time we will start using these names in subsequent cmdlets and automations.