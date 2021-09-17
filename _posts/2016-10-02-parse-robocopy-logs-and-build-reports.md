---
layout: post
title: Parse robocopy logs and build reports
date: 2016-10-02 21:22:40.000000000 +03:00
categories: [PowerShell, Robocopy]
tags: [PowerShell, Robocopy]
---
Quick way to parse number of robocopy logs and produce some report files based on them. Robocopy  logs are written with LOG+ option, so parser is able to see start and end time for each job, detect failed jobs and duration of the job.
Here is the example

{% gist ba2baf5e1ecf6b28764611ad99b5dfd8 robocopyParse.ps1 %}

Try and enjoy!