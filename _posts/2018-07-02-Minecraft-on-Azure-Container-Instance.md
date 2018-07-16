---
layout: post
title:  "Time to play: Minecraft on Azure Container Instance"
categories:  ['PowerShell', 'Azure']
tags:  ['PowerShell', 'Azure', 'Powershell Core']
date:  2018-07-02 07:38:04 +03:00
beginps: '<div class="syn_wrapper"><pre class="brush: powershell;">'
endps: '</pre></div>'
---

Hello colleagues!
Sometimes it is good to spend some time playing with kids. So I play Minecraft with my kid from time to time. And so I decided to try it in Azure Container Instances. What i want to do is, i want to see if it is cheaper than running it on a VM or not. Another thing i wanted to try is to configure all of this from my Mac. Actually my OS of choice is Windows, but this time i tried to use Mac.
So what we are going to look at today?

- Create and run a container in Azure Container Instance
- Attach a persistent storage to it to store Minecraft's state
- Copy state from the existing Minecraft server running Ubuntu, to the Azure File Share

<!--more-->

Well, frankly i spent two hours trying this and that. However most of the time i tried to copy proper set of files. But, fist thing first.

I use [itzg/minecraft-server](https://hub.docker.com/r/itzg/minecraft-server/) as an image for my server. And previously i ran it on a VM in Azure. So, as far as i decided to move, it was a good idea to copy everything related to my World to a new place, so we can continue in the same World. In Azure Container Instances there is an option to mount Azure File Share as a persistent storage to a container. In my case i needed to copy my World to Azure Storage Files, and then mount this share to my container.
What to do, to copy some files from a Linux box to Azure Storage, you may ask. Fist of all you need to create a resource group and put your storage account there. Well, then [azcopy](https://docs.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-linux) is a tool you use to do this. You need to have your storage access key, but you can easily find it on Azure Portal.

![storagekey](/images/posts/2018-07-02-Minecraft-on-Azure-Container-Instance/storageKey.png)

In addition to that you need to create an Azure File Share in your Storage Account, and get a link to it. For that you click "File" ling on the Portal

![filecreation1](/images/posts/2018-07-02-Minecraft-on-Azure-Container-Instance/fileShareCreate.png)

and the add the share

![filecreation2](/images/posts/2018-07-02-Minecraft-on-Azure-Container-Instance/fileShareCreate02.png)

Next, you get the URL to your share:

![fileshareurl](/images/posts/2018-07-02-Minecraft-on-Azure-Container-Instance/url.png)

And the command will look like

![azcopy](/images/posts/2018-07-02-Minecraft-on-Azure-Container-Instance/azCopy.png)

and in few seconds you have your files uploaded to the share.

Great, now what is left is to create your container in Azure. This is simple:

{{ page.beginps }}
$secpasswd = ConvertTo-SecureString "<your key>" -AsPlainText -Force
$mycred = New-Object System.Management.Automation.PSCredential ("<your storageacct name>", $secpasswd)
New-AzureRmContainerGroup -ResourceGroupName minecraft `
                          -Cpu 2 -MemoryInGB 4 -Name <yourname> `
                          -Image "itzg/minecraft-server" `
                          -AzureFileVolumeShareName <yourname> `
                          -AzureFileVolumeAccountCredential $mycred  `
                          -AzureFileVolumeMountPath "/data" `
                          -OsType Linux -IpAddressType Public -Port @(25565) `
                          -DnsNameLabel <yourname> 
{{ page.endps }}

this looks something like this

![create container](/images/posts/2018-07-02-Minecraft-on-Azure-Container-Instance/createContainer.png)

Note, there is a DNS name assigned to this container group, so we can reference it in our Minecraft client.

This is it, after few minutes your container is up and running with whole my World in it!