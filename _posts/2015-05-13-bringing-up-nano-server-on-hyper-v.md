---
layout: post
title: Bringing up Nano Server on Hyper-V
date: 2015-05-13 19:52:50.000000000 +03:00
categories: [NanoServer]
tags: [NanoServer]
---
On the MS Ignite new thing, [Nano Server](http://blogs.technet.com/b/windowsserver/archive/2015/04/08/microsoft-announces-nano-server-for-modern-apps-and-cloud.aspx) was announced. The idea is to get smallest footprint for the Windows Server ever. Now it is possible to try it. In general it is quite straightforward. First you need to download the technical preview of new Server from [here](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-technical-preview). 

>There you will find the gate to hell, opened before you. You must find the courage to step through that gate …" (c) oops ). 
No, not that bad.

After you have downloaded the image you can mount it and find a folder called NanoServer. There will be the wim image of the server and some additional folder with packages wich can be additionally added to the new installation. Next you need to follow the link [here](http://www.aka.ms/nanoserver) and read the documentation. What I did is just wrapped the doc into a small script which shows the steps in brief

```powershell
$dism = 'D:\temp\NewDism\dism.exe'
$imageFile = 'D:\VMs\Virtual Hard Disks\nanoServerBaseLine2.vhd'
$mountDir = 'D:\temp\mountdir'
$convert = 'D:\Temp\Convert-WindowsImage.ps1'
$convert -SourcePath E:\NanoServer\NanoServer.wim -VHDPath $imageFile -VHDFormat VHD -Edition 1
$dism /Mount-Image /ImageFile:$imageFile /Index:1 /MountDir:$mountDir
$dism /Add-Package /PackagePath:E:\NanoServer\Packages\Microsoft-NanoServer-Guest-Package.cab /Image:$mountdir
$dism /Image:$mountdir /Apply-Unattend:D:\VMs\Unattend.xml
md $mountDir\windows\panther
copy D:\VMs\Unattend.xml $mountDir\Windows\panther
$dism /Unmount-Image /MountDir:$mountdir /Commit
New-VM –Name testNanoVM –MemoryStartupBytes 1GB –VHDPath $imageFile -SwitchName Internal | Start-VM
```