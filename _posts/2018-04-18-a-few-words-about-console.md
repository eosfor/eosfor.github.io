---
layout: post
title:  a few words about console
categories:  ['Common']
tags:  ['Console','OpenSSH']
date:  2018-04-18 06:10:20 +03:00
beginps: '<div class="syn_wrapper"><pre class="brush: powershell;">'
endps: '</pre></div>'
---

Hello colleagues. Until very recently i've been strongly a Windows guy. However, fortunately or not, the World has changed. As part of this change I was asked to do some stuff on Linux. So a few outcomes of this work. 

First of all I discovered a really cool tool - [ColorTool](https://github.com/Microsoft/Console/tree/master/tools/ColorTool).  What it does is, it can make your windows console colorful! nd it is pretty nice.

Second nice thing is [OpenSSH-Win32](https://github.com/PowerShell/Win32-OpenSSH/releases). This repo provides a lot of tools to work natively with ssh/sftp/scp. So I decided to try using these tools instead of putty. And they were pretty good, so far.

So basically the way I do this is pretty simple. I got a small bat file which start a console and colorizes it in the way I like. Next step for me is just to cd into a Win32-OpenSSH folder and start from there.