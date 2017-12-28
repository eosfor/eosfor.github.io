---
layout: post
title: How to bring up Windows 2012 R2 in HyperV without installing it
date: 2015-05-08 15:49:17.000000000 +03:00
type: post
categories: [Powershell, Automation]
tags: [ISO, VHD, Powershell, Automation]
---
Hey, [here is a great stuff](https://gallery.technet.microsoft.com/scriptcenter/Convert-WindowsImageps1-0fe23a8f) out there. This script is able, so to speak, to convert an ISO file into VHD(x) file, that can be easily attached to the VM. Basically you take ISO, convert it, attach VHD to a VM and voila, you are done. So what to do to achieve this? Easy, just download the script! And after that just:

```powershell
.\Convert-WindowsImage.ps1 -source D:\X64FRE_SERVER_EVAL.ISO -Edition ServerDataCenterEval -VHDPath 'D:\VMs\Virtual Hard Disks\CONTOSO-DC1.vhdx' -VHDFormat vhdx -VHDPartitionStyle GPT
```

![img](/images/posts/oldposts/2015-05-08-18_41_49-administrator_-windows-powershell.png)