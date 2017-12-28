---
layout: post
title: Some new syntax in Powershell
date: 2015-02-27 18:28:02.000000000 +03:00
categories: [PowerShell]
tags: [PowerShell]
---
Now sure when this happened, but looks like we have new syntax for defining arrays. It looks like below:

```powershell
$a = $(1,2,3)
```

What is this, what is the result here. Lets try some new features of [WMF 5 February preview](http://www.microsoft.com/en-us/download/details.aspx?id=45883). Just do couple of things

- Download and install the preview
- Start the PowerShell with admin credentials
- Run the following command. This going to download the module from PowerShell repository from [here](https://www.powershellgallery.com)

```powershell
Install-Module -Name PowerShellCookbook
```

![img](/images/posts/oldposts/2015-02-27-20_43_18-administrator_-windows-powershell.png)

At this point we have installed the [PowerShellCookbook](https://www.powershellgallery.com/packages/PowerShellCookbook/1.3.6) module and we are interested in Show-Object cmdlet from it. We can just pipe pass the variable to it and see the below mega-thing!

```powershell
Show-Object $a
```

![img](/images/posts/oldposts/array.png)

Really cool. Here we see it is an array
