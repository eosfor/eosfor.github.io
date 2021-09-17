---
title:  "Welcome to Jekyll!"
date:   1981-08-02 15:04:23
categories: [jekyll]
tags: [jekyll]
---
Here is the first PowerShell post I've made

Jekyll also offers powerful support for code snippets:

``` powershell
get-command get-item
```

<h1 style="border-bottom: 2px solid black;">powershell</h1>
<pre class="brush: powershell;">
$request = [System.Net.WebRequest]::Create(&#34;http://www.somelocation.com/testlink.aspx&#34;)
$response = $request.GetResponse()
$requestStream = $response.GetResponseStream()
$readStream = new-object System.IO.StreamReader $requestStream
new-variable db
$db = $readStream.ReadToEnd()
$readStream.Close()
$response.Close()
</pre>

HO HO HO!