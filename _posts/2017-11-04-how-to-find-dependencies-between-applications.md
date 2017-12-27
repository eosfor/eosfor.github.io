---
layout: post
title: How to find dependencies between applications
date: 2017-11-04 16:10:28.000000000 +03:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- PowerShell
tags: []
meta:
  _rest_api_published: '1'
  _rest_api_client_id: "-1"
  _publicize_job_id: '11075911916'
  _publicize_failed_18229674: O:13:"Keyring_Error":2:{s:6:"errors";a:1:{s:21:"keyring-request-error";a:1:{i:0;a:6:{s:7:"headers";O:42:"Requests_Utility_CaseInsensitiveDictionary":1:{s:7:"
author:
  login: eosfor
  email: eosfor@gmail.com
  display_name: eosfor
  first_name: ''
  last_name: ''
---
<p>Hello colleagues. As part of migrations to the cloud most common question is - how to map existing application landscape? How to find all relations of the application being moved? How to see which systems it communicates to?</p>
<p>Actually I already blogged about this some time ago but this time example is a bit more clear. We need to do the following steps to graph our environment:</p>
<ol>
<li>Enable and collect Windows Firewall logs  for all systems in the environment</li>
<li>Download and install <a href="https://www.powershellgallery.com/packages/PSQuickGraph/1.1">PSQuickGraph</a></li>
<li>Analyze the logs</li>
</ol>
<p>For the second step here is the simplest <a href="https://gist.github.com/eosfor/a29f53052c7bdfc98339e630149cccdb">example</a></p>
<p>And here is what we can get out of it.</p>
<p><img class="alignnone size-full wp-image-527" src="{{ site.baseurl }}/assets/apprelgraph.png" alt="apprelgraph" width="1197" height="850" /></p>
