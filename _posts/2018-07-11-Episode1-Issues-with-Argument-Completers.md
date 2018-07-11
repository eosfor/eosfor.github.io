---
layout: post
title:  '06. Episode 1: There are some "peculiarities" with argument completers, however'
categories:  ['PowerShell']
tags:  ['PowerShell']
date:  2018-07-11 01:37:23 +03:00
beginps: '<div class="syn_wrapper"><pre class="brush: powershell;">'
endps: '</pre></div>'
---

Testing some stuff with argument completers [in the previous article](./2018-07-08-Episode1-lets-talk-to-cmdb-and-autocomplete!.md) I found some, say, "peculiarities". So lets go through them. Link to our samples github repo is [here](https://github.com/eosfor/cloudmgmt)

# Review the existing options

Well, first of all we touched some options, we can use to make our completers. They are:

- [[ValidateSet()]](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_advanced_parameters?view=powershell-6#validateset-attribute) attribute
- [TabExpansionPlusPlus](https://github.com/lzybkr/TabExpansionPlusPlus) module
- [[ArgumentCompletions()]](https://docs.microsoft.com/en-us/dotnet/api/system.management.automation.argumentcompletionsattribute?view=pscore-6.0.0) attribute
- [[ArgumentCompleter()]](https://docs.microsoft.com/en-us/dotnet/api/system.management.automation.argumentcompleterattribute?view=pscore-6.0.0) attribute

I played a bit with TabExpansionPlusPlus and [ArgumentCompleter()] attribute and here are some findings I got.

# [ArgumentCompleter()] attribute

## Using it with a scriptblock

First of all if we look at the code [here](https://github.com/PowerShell/PowerShell/blob/58e9b4969aa9c3c6c288c30293ef3af274e23d55/src/System.Management.Automation/engine/CommandCompletion/ExtensibleCompletion.cs#L23) we will see that attribute can take two types of parameters. First is if type "type", which in fact should be derived from [IArgumentCompleter]. And second is a script block. We were using the second option in our previous example.

But here is what I found. In our test we will remove everything related to TabExpansionPlusPlus from our code and do our test. Here is what we are going to call

```powershell
function Get-CMDBdata {
    [CmdletBinding()]
    param (
        [ArgumentCompleter({& CMDBBUCompleter3 @args})]
        $BU,
        [ArgumentCompleter({& CMDBDEPTCompleter2 @args})]
        $DEPT
    )

    process {
         & $cmdbFunc
    }

}
```

And here is how it looks like

![completers do not work](images/posts/2018-07-11-Episode1-Issues-with-Argument-Completers/doNotWork.gif)

As you can see, cmdlet itself works fine, it returns our data. But completer does not work! If I return TabExpansionPlusPlus into the picture it starts working. But why? it calls the same function as TabExpansionPlusPlus!

Well, I played around with it a little bit. Here are some findings. If we look at the ```function:\``` drive in a global scope, here what we can see:

![functions global](images/posts/2018-07-11-Episode1-Issues-with-Argument-Completers/functions1.png)

there is not "completer" functions there. Ok, lets look at what module sees internally. For that lets set a breakpoint inside one of our functions and see this drive from there. We can do this right in the console by using ```Set-PSBreakpoint``` cmdlet, like this.

![set ps breakpoint](images/posts/2018-07-11-Episode1-Issues-with-Argument-Completers/psbreakpoint.png)

Now lets see the drive:

![drive from function](images/posts/2018-07-11-Episode1-Issues-with-Argument-Completers/psbreakpointdrive.png)

Yikes, functions are there! Well, lets now try to put these functions to a global scope. For that we can change our ```Export-ModuleMember -Function *-*``` call in cloudmgmt.psm1 to export all functions, but not just those with dash ```-``` in their name, like below, and try again

```powershell
# Export-ModuleMember -Function *-*
Export-ModuleMember -Function *
```

And here is what we've got

![working via publishing](images/posts/2018-07-11-Episode1-Issues-with-Argument-Completers/globalScopeWorking.gif)

Basically we see that everything starts working again. When we hit <TAB> it takes some time to call the function and populate the cache and then it start completing parameters. So what is our global ```function:\``` dive now?

![global drive after exporting](images/posts/2018-07-11-Episode1-Issues-with-Argument-Completers/exportedGlobally.png)

Functions exported to a global scope and ```[ArgumentCompleter()]``` started working again.

So the bottom line here - you should export your completers if you what your ```[ArgumentCompleter()]``` working. Which is not good, I think. IMHO I'd to hide this complexity inside module, so I think it is time to raise an issue on github.

## Using it with a type

Another option to initialize ```[ArgumentCompleter()]``` is using type, derived from [IArgumentCompleter]. Lets look at this option. Before testing we get back to ```Export-ModuleMember -Function *-*```. Now what we need to do, is to declare and implement our type, like this:

```powershell
class CMDBBUCompleter4 : IArgumentCompleter
{
    [IEnumerable[CompletionResult]] CompleteArgument(
        [string] $CommandName,
        [string] $parameterName,
        [string] $wordToComplete,
        [CommandAst] $commandAst,
        [IDictionary] $fakeBoundParameters)
    {
        $resultList = [List[CompletionResult]]::new()

        (& $Global:cmdbFunc) | ? PartitionKey -EQ "BU" |            #<- pay attention to this line
        Select-Object code, name -Unique |
        ? {$_.name -like "*$wordToComplete*"} | % {
            $resultList.Add([CompletionResult]::new($_.code, $_.name, "ParameterValue", 'BU'))
        # New-CompletionResult -CompletionText $_.code -ToolTip 'BU' -ListItemText $_.name
     }

     return $resultList
    }
}
```

It got everything we used previously, the same set of parameters, the same result type, the same body. Well, almost the same. If we try to use our variable, instead of a direct call to the CMDB interface, we get PSAnalyzer error message, like below.

![cant use a variable](images/posts/2018-07-11-Episode1-Issues-with-Argument-Completers/completerClassProblem.png)

To work this around we can use scope modifier

Good, so we updated our code, we returned ```Export-ModuleMember``` as it was before, and added another class. Now we need to update our cmdlet in a proper way, like below, and try

```powershell
function Get-CMDBdata {
    [CmdletBinding()]
    param (
        [ArgumentCompleter([CMDBBUCompleter4])] # <- here is the update
        $BU,
        [ArgumentCompleter({& CMDBDEPTCompleter2 @args})]
        $DEPT
    )

    begin {
    }

    process {
         & $cmdbFunc
    }

    end {
    }
}
```

And the result is - it is working. Even with functions not in global scope.

![completer with class](images/posts/2018-07-11-Episode1-Issues-with-Argument-Completers/completerWithClass.gif)

So, at the moment it looks like we should stick to using this attribute + export all functions globally. But, will see ...

This is it for this article, and lets move forward. I hope, next time we will complete our name generation at last!