---
layout: post
title:  'Episode 1: Lets talk to CMDB at last!'
categories:  ['PowerShell','Azure','Tab']
tags:  ['PowerShell','Azure']
date:  2018-07-08 02:55:44 +03:00
beginps: '<div class="syn_wrapper"><pre class="brush: powershell;">'
endps: '</pre></div>'
---

Hello colleagues, we continue building our Azure management module. Next step for us is to complete our CMDB interface and make a cmdlet, which will be used by rest of our code to talk to CMDB. Lets do this, but first we need to recall our goals for it.

# Goals

We got the following expectations from this command

- The command should not depend on implementation of the CMDB-related part
- We want to be have auto-completion for parameter values

So let's first focus on the first goal.

# Making cmdlet independent

At this point we will talk about few things

- First - [```Get-Command```](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command?view=powershell-6) cmdlet
- Second - Call operator ```&```

Well, in general, how can we make a cmdlet independent of underlying cmdlets. I mean, imagine the following scenario. You got ```Get-MyData``` cmdlet which in turn should talk to different data sources using different cmdlets, like ```Get-MyDataFromMSSQL``` or ```Get-MyDataFromCosmosDB```. How can we make it so that we dont need to change it's code in case we need to switch from ```Get-MyDataFromMSSQL``` to ```Get-MyDataFromCosmosDB```, for example? And, we want this to be configurable, to the certain extent. We want to be able get back, or change to something else. So what we do?

What do I typically use to resolve this is the fact that commands are objects and you can store then as variables. And call them when needed. And you can change the value of the variable when you need it, like this

![call operator](/images/posts/2018-07-08-Episode1-lets-talk-to-cmdb-and-autocomplete/callOperatorSample.gif)

As you can see, you can store a command into a variable and then call it!

So first of all we declare this variable and put is into the psm1 file, like this. This will make this variable available to the whole module. Let it bs Global so far so we can debug or look at this closer.

```powershell
Get-ChildItem $PSScriptRoot\*.ps1 -Recurse | ? {$_.Directory -notlike "*test*"}  | % {
    . $_.FullName
}

$Global:cmdbFunc = (Get-Command Get-AzureCMDBData)

Export-ModuleMember -Function *-*
```

With this we achieve two things. First is that, we make this kind of "configurable", because if we need to change our CMDB, we will just need to change the command in this line from ```Get-AzureCMDBData``` to something else. And this will change behavior of our script entirely. Next thing is that, we are now "decoupled" from the implementation of CMDB part. We just need to call the cmdled stored in the variable, and we do not really care what is stored there. The only requirement will be - something we assign to a variable should return results of the same structure (array of ```[pscustomobject]```), and take the same arguments.

Now we can use it like this:

```powershell
function Get-CMDBdata {
    [CmdletBinding()]
    param (
    )
    process {
         & $cmdbFunc
    }
}
```

So at this point everything else inside our module, that needs any kind of data from CMDB, should call this function ```Get-CMDBdata```. This function is now abstracted from CMDB details and and if we change CMDB all the tuff in our module remain the same. We wont need to change any code of any other cmdlet. Here is how it looks like in the console

![cmdb call](/images/posts/2018-07-08-Episode1-lets-talk-to-cmdb-and-autocomplete/cmdbCall.gif)

So, as an interim result what we managed to achieve? We now have a replaceable piece which talks to our sample CMDB stored in Azure Table. In addition to the possibility to query data, it also provides simple cache, which is transparently used by underlying cmdlets and initialized lazily. All other cmdlets in our module receive benefits from using this cache without even knowing about it. And all of this is done on pure PowerShell Core and is cross-platform. Here is how it looks on [WSL Ubuntu](https://docs.microsoft.com/en-us/windows/wsl/about).

![cmdb call WSL](/images/posts/2018-07-08-Episode1-lets-talk-to-cmdb-and-autocomplete/cmdbCallWSL.gif)

# Adding auto-completion

Yeah, everything looks cool so far. Next thing in our task list is auto-completion. We want two thing:

- Auto-complete BU and DEPT parameters for the ```Get-CMDBdata``` so that users wont make mistakes when calling it from PowerShell manually
- DEPT parameter completion should rely on parameter provided for BU, and show only departments of chosen BU

## Parameter completion in general

There are a lot of options to help users of your cmdlets to avoid mistakes. The simplest one is an ```[ValidateSet()]``` attribute. Like this

![validateSet](/images/posts/2018-07-08-Episode1-lets-talk-to-cmdb-and-autocomplete/validateSet.gif)

However, this method is "static". You can't change the list on the fly. There are two options to make this more "dynamic". First option is [lzybkr/TabExpansionPlusPlus](https://github.com/lzybkr/TabExpansionPlusPlus) module. This module works for PowerShell 5 and PowerShell Core. This is kind of a classic way to do so. The modern way is using [ArgumentCompleter attribute](https://docs.microsoft.com/en-us/dotnet/api/system.management.automation.argumentcompleterattribute?view=powershellsdk-1.1.0). This works on PowerShell Core only (but i did not test it, on PS5), and looks more, say, natural. Lets quickly look at both options.

### Old school: TabExpansionPlusPlus

It is more or less straightforward. First you declare a function, you want to call on completion event. It takes specific set of parameters, and gets called at the moment you hit <TAB> or when IntelliSense pops up. It performs some magic, and should return a set of objects of specific type. These objects in turn generated by ```New-CompletionResult``` cmdlet. It looks like this

```powershell
function CMDBBUCompleter {
    param($commandName, $parameterName, $wordToComplete, $commandAst, $fakeBoundParameter)
    (&$cmdbFunc) | ? PartitionKey -EQ "BU" |
        Select-Object code, name -Unique |
        ? {$_.name -like "*$wordToComplete*"} | % {
            New-CompletionResult -CompletionText $_.code -ToolTip 'BU' -ListItemText $_.name
     }
}
```

After this you need to register this function with the cmdlet you want to auto-complete parameter values for. Something like this:

```powershell
TabExpansionPlusPlus\Register-ArgumentCompleter -CommandName Get-CMDBdata `
    -ParameterName BU `
    -ScriptBlock $Function:CMDBBUCompleter `
    -Description "BU"
```

Where ```-CommandName``` specifies the cmdlet we want to auto-complete for ```-ParameterName``` is the name of a paramter we auto-complete, ```-ScriptBlock``` is something we want to attach, it is basically our code block. In simple terms this is it

### Modern: [ArgumentCompletions] attribute

This attribute is similar to ```[ValidateSet()]```, but a bit different. ```[ValidateSet()]``` enforces your possible parameter values to a some strict set. If you pass a value which is not in the set it fails. But what if we need to give some list of possible completions, but do not want to restrict the list completely? This is what ```[ArgumentCompletions()]``` attribute is for. Here is the difference:

Some code

```powershell
function foo(){
    [cmdletbinding()]
    param(
        [ValidateSet("a","b")]
        $p
    )

    $p
}

function foo2(){
    [cmdletbinding()]
    param(
        [ArgumentCompletions("a","b")]
        $p
    )

    $p
}
```

and it's output

![ArgumentCompletions](/images/posts/2018-07-08-Episode1-lets-talk-to-cmdb-and-autocomplete/autocompleteAttr.png)

As you can see, the first function fails when you give it a value, which is not in the set.

But this option as not "dynamic" too. We can't change this list on the fly.

### Modern: [ArgumentCompleter] attribute

The modern, dynamic way looks like this. We still need to define a function, but in order to attach it to a parameter we do something like:

```powershell
function Get-CMDBdata {
    [CmdletBinding()]
    param (
        [ArgumentCompleter({CMDBBUCompleter @args})] # <- here it is!
        $BU,
        $DEPT
    )
     process {
         & $cmdbFunc
    }
```

We provide the ```[ArgumentCompleter]``` attribute to a parameter, and pass a scriptblock to it. This scriptblock in turn, calls our function and takes ```@args```, which is basically a set of parameters passed to the scriptblock by the PowerShell, see [PowerShell splatting](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_splatting?view=powershell-6) for more details about the syntax. To me it looks more natural. So in our scenario we will follow this way.

## Picking up the value from the previous command

What's next? Well, what is left here is an option to filter possible parameter values of one of the parameters based on values provided to another parameter. Say, we provided the value 'PDN' to a ```-BU``` parameter, so in the list of possible values of the ```-DEPT``` parameter we should be able to pick only those, which are of specified BU. And PowerShell provides a possibility for that! The very last parameter of the function we use is ```$fakeBoundParameter```, which is a hash table. This parameter contains "bound parameters", as they will look like after they are actually bound. And we can use it to peek into what will be passed to a previous parameter, at the moment we call a cmdlet. And based on this value we can do some logic. Here is the example:

```powershell
function CMDBDEPTCompleter2 {
    param($commandName, $parameterName, $wordToComplete, $commandAst, $fakeBoundParameter)

    $buParamValue = $fakeBoundParameter.BU
    $BUs = (& $cmdbFunc) | ? PartitionKey -EQ "DEPT" | ? BU -EQ $buParamValue | Select-Object code, name -Unique

    $BUs | ? {$_.name -like "*$wordToComplete*"} | % {
        New-CompletionResult -CompletionText $_.code -ToolTip 'DEPT' -ListItemText $_.name
    }
}
```

Here we pick a value of a ```-BU```parameter, and based on it we can filter and find values we need. Please also pay attention to the fact, that we use our ```$cmdbFunc``` variable, so completers will continue work when we change a CMDB provider!

Well, there is another option to do this:

```powershell
function CMDBDEPTCompleter {
    param($commandName, $parameterName, $wordToComplete, $commandAst, $fakeBoundParameter)
    $buparamindex = $commandAst.CommandElements.IndexOf(($commandAst.CommandElements | ? ParameterName -EQ "BU")) #<- finding an index of a value
    $buParamValue = $commandAst.CommandElements[$buparamindex + 1].Value  #<- extracting a value
    $BUs = (& $cmdbFunc) | ? PartitionKey -EQ "DEPT" | ? BU -EQ $buParamValue | Select-Object code, name -Unique

    $BUs | ? {$_.name -like "*$wordToComplete*"} | % {
        New-CompletionResult -CompletionText $_.code -ToolTip 'DEPT' -ListItemText $_.name
    }
}
```

Here we follow a bit different strategy. Parameter ```$commandAst``` contains [Abstract Syntax Tree (AST)](https://github.com/dfinke/PowerShellASTDemo), which basically some kind of a structure, containing parsed PowerShell/PS command line tokens. So we can extract a value of a command from this parsed data. First of all we look for an index of ur parameter and then extract it's value. Result looks like to be the same, but the way with ```$fakeBoundParameter``` looks more attractive to me.

So, at the end of the day, here is how the resulting command is going to look like:

```powershell
function Get-CMDBdata {
    [CmdletBinding()]
    param (
        [ArgumentCompleter({CMDBBUCompleter3 @args})] # <- first argument completer (for BUs)
        $BU,
        [ArgumentCompleter({CMDBDEPTCompleter2 @args})] # <- second argument completer (for DEPTs)
        $DEPT
    )
     process {
         & $cmdbFunc # <- will add filtering next time ;)
    }
}
```

And this is how it works

![working sample](/images/posts/2018-07-08-Episode1-lets-talk-to-cmdb-and-autocomplete/completionWorking.gif)

So, this is it for today. Hope this was useful. Next time we will at last implement our naming generation cmdlet and extend our goals for this automation module!