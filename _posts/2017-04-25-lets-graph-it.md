---
layout: post
title:  Lets graph it!
date:   2017-04-25 22:06:50.000000000 +03:00
categories: [powershell]
tags: [powershell]
---

So, after some time my  friends and I released the first version on the [PSQuickGraph](https://www.powershellgallery.com/packages/PSQuickGraph/1.0) module. It is also available on [GitHub](https://github.com/eosfor/PSGraph). It is  to be able to analyze dependencies in a scripted way.

Let's give it a try. First you need to do is to install it. To do it, just start PowerShell and issue the following command, which  installs  the module from PS  gallery

```
Install-Module PSQuickGraph -Scope CurrentUser
```

Next thing you need to do is to install  [graphviz](http://graphviz.org/). It is a command line tool which takes the text file in a special format and makes a picture from it. You can take it from [here](http://graphviz.org/Download_windows.php) and simply unzip into a folder. In my case I use c:\temp\graphviz folder. Once it is there you can do some magic.

First -  start PowerShell and say:

```powershell
Import-Module PSQuickGraph
```

Now just open [this link](https://raw.githubusercontent.com/eosfor/PSGraph/master/PSGraph/processTreeSample.ps1) and copy the contents to the PowerShell window. And in a second  you will see a your first graph made using PowerShell.

You can do a lot more with it,
Enjoy!