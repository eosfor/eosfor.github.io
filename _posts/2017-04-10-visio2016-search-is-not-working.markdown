---
title:  "Visio 2016 search is not working"
date:   2017-04-10 23:18:24.000000000 +03:00
categories: [common]
tags: [visio,troubleshooting]
---
Actually I have no idea how this works, probably I need to investigate deeper, but now I have not enough time tor that. So the issue was with the search, like [here](https://superuser.com/questions/208395/how-do-i-fix-the-error-visio-cannot-provide-fast-search-results) or [here](https://answers.microsoft.com/en-us/msoffice/forum/msoffice_visio-mso_windows8/visio-cannot-provide-fast-search-results/9fdc81da-8f56-4ba4-9ff7-44cc0865787f).
Search simply was not working. I ran win10 + Visio 2016 pro, all x64. I downloaded Azure and AWS  stencils, put them  to My Shapes folder under Documents and it did not work.

I spent huge amount of time poking around, I was rebuilding indexes, restarting services, copying files. I even looked into the index with PowerShell. Files were there but Indexing did not work. And as last resort I just copied and pasted all the files inside of the same directory:
C:\Program Files\Microsoft Office\root\Office16\Visio Content\1033. And suddenly it worked!

So the workaround was to:
 - Copy all stencil files you need to a C:\Program Files\Microsoft Office\root\Office16\Visio Content\1033
 - Select them all, copy (CTRL+C) and paste (CTRL+V) them once again in the same directory

And they immediately appeared in the search! Don't ask me why.

![Img]({{ site.baseurl }}/images/posts/oldposts/visiofilescopied.png)
![Img]({{ site.baseurl }}/images/posts/oldposts/visiofilescopied2.png)

PS. 

One more location i've found is here C:\Program Files\MSOffice\Office16\Visio Content\1033

But still there are some issues. Just realized that not all files have been indexed. So, give it a try anyway!