---
title:  "Visio 2016 search is not working"
date:   2017-04-10 23:18:24.000000000 +03:00
categories: [common]
tags: [visio,troubleshooting]
---
<p>Actually I have no idea how this works, probably I need to investigate deeper, but now I have not enough time tor that. So the issue was with the search, like <a href="https://superuser.com/questions/208395/how-do-i-fix-the-error-visio-cannot-provide-fast-search-results">here</a> or <a href="https://answers.microsoft.com/en-us/msoffice/forum/msoffice_visio-mso_windows8/visio-cannot-provide-fast-search-results/9fdc81da-8f56-4ba4-9ff7-44cc0865787f">here</a>. Search simply was not working. I run win10 + Visio 2016 pro, all x64. I downloaded Azure and AWS  stencils, put them  to My Shapes folder under Documents and It sis not work.</p>
<p>I spent huge amount of time poking around, I was rebuilding indexes, restarting services, copying files. I even looked into the index with PowerShell. Files were there but Indexing did not work.</p>
<p>And as last resort I just copied and pasted all the files inside of the same directory:<br />
C:\Program Files\Microsoft Office\root\Office16\Visio Content\1033. And suddenly it worked!</p>
<p>So the workaround was to:</p>
<ol>
<li>Copy all stencil files you need to a C:\Program Files\Microsoft Office\root\Office16\Visio Content\1033</li>
<li>Select them all, copy (CTRL+C) and paste (CTRL+V) them once again in the same directory</li>
</ol>
<p>And they immediately appear in the search! Don't ask me why.</p>
<p><img class="alignnone size-full wp-image-239" src="{{ site.baseurl }}/images/posts/visiofilescopied.png" alt="visioFilesCopied" width="1016" height="600" /></p>
<p><img class="alignnone size-full wp-image-247" src="{{ site.baseurl }}/images/posts/visiofilescopied2.png" alt="visioFilesCopied2" width="278" height="435" /></p>
