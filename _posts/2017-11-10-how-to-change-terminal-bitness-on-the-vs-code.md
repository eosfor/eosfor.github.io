---
layout: post
title:  How to change bitness of the terminal in VSCode
categories: [PowerShell]
tags: [PowerShell]
date: 2017-11-10 15:48:38.000000000 +03:00
---
I suddenly realized that VS Code start 32bit PowerShell host as a terminal. For some cases it does not work well so I decided to try to change it to run 64bit process instead. Here is how it can be done:

![Img]({{ site.baseurl }}/images/posts/2017-11-10-how-to-change-terminal-bitness-on-the-vs-code/vcsodepsx64.png)

On the step 1 you need to change SysWOW64 to System32, and then restart the terminal by pressing CTRL+SHIFT+P and typing reload.

![Img]({{ site.baseurl }}/images/posts/2017-11-10-how-to-change-terminal-bitness-on-the-vs-code/reload.png)

After that you just use [System.Environment]::Is64BitProcess snippet to check if everything is ok.