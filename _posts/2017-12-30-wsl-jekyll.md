---
layout: post
title: "WSL, Jekyll and crazy cool things you can now do on Windows"
categories:  ['PowerShell']
tags:  ['PowerShell']
date:  2017-12-30 08:31:14 +03:00
beginps: '<div class="syn_wrapper"><pre class="brush: powershell;">'
endps: '</pre></div>'
images: "/images/posts"
excerpt_separator: <!--more-->
---
Hello colleagues. Today i'm going to try to show how i managed to setup blog on [Github Pages](https://pages.github.com) using WSL on Windows.
<!--more-->

Let's start!
First of all we need to install WSL. I choose to install Ubuntu, as, i don't know, just because. I'm Windows guy, so it doesn't make any difference to me, which Linux distribution to use. If so - why i even going to use WSL. First of all because of Jekyll. On it's main web page i see few lines of instructions for Linux. So, it seems to be easy to do using Linux tools and approaches. In addition to that i what to try WSL.

## Installing WSL

For this i need to go to Microsoft Store, search for Ubuntu and click "Install". 

![WSLInstallation]({{page.images}}/wslUbuntuInstallation.png)

After some time we get a message saying that it is installed.

![WSLInstalled](http://content.screencast.com/users/eosfor/folders/Snagit/media/6359645d-f0ab-4707-924b-4087bd9efcf9/12.30.2017-20.53.png)

In fact it is not. When we start Ubuntu from "Start" menu it shows the console window where we first, need to wait a bit more, and then we need to put some additional info - username and password. And after all these steps voila - we got "Linux" installed.

![WSLInstalled2](https://content.screencast.com/users/eosfor/folders/Snagit/media/5f55ce2a-02d0-40e8-9205-a776f3fa7786/12.30.2017-21.29.png)

## Jekyll installation

So far so good. Lets now install couple of Linux-based components. From the Jekyll installation guide i got that there are following requirements:

- GNU/Linux, Unix, or macOS
- Ruby version 2.2.5 or above, including all development headers RubyGems
- GCC and Make (in case your system doesn’t have them installed, which you can check by running gcc -v and make -v in your system’s command line interface)

Ok, let's assume we have some Linux. Let's go and install some Ruby. But first ``` sudo apt update ```, and then, as described on [docs page](https://www.ruby-lang.org/en/documentation/installation/#apt), ``` sudo apt-get install ruby-full ```

![InstallingRubyFull](https://content.screencast.com/users/eosfor/folders/Snagit/media/e2d03ec4-d9cb-46b2-8b08-5f5d787c4d73/12.30.2017-21.27.png)

In addition to that seems they want development headers to be installed. Let's do that as well, ``` sudo apt-get install ruby-dev ```, but it seems to be already there:

![InstallingRubyDev](https://content.screencast.com/users/eosfor/folders/Snagit/media/106fd7ec-9217-4dbb-8f7e-aefa6aa7a0bc/12.30.2017-21.26.png)

Let's also install gcc and make, as Jekyll requires:

{% highlight bash linenos %}
sudo apt install gcc
sudo apt install make
{% endhighlight %}

Ok, done

![InstalledGCCandMake](https://content.screencast.com/users/eosfor/folders/Snagit/media/146a72ed-1c8f-495b-9be8-a88bcaae1710/12.30.2017-21.26.png)

- [x] GNU/Linux, Unix, or macOS
- [x] Ruby version 2.2.5 or above, including all development headers RubyGems
- [x] GCC and Make

Good, now let's follow the book and install Jekyll itself ``` sudo gem install jekyll bundler ```. Ok, done. Let's check it

{% highlight bash linenos %}
sudo jekyll new my-awesome-site
cd my-awesome-site/
sudo bundle exec jekyll serve
{% endhighlight %}

And what we see?

![InstalledGCCandMake](https://content.screencast.com/users/eosfor/folders/Snagit/media/fd3debad-19ae-4450-837c-478ca2726eef/12.30.2017-21.36.png)

Crazy cool (c)!! This means that WSL works perfectly! It can run Ruby, and even serve web sites!

## Jekyll-uno theme

Great, now we can try to build some more nice-looking site. As i see a lot of good people out ther use [Jekyll-uno](https://github.com/joshgerdes/jekyll-uno) theme. So let's go for it. In addition to that let's put it to a Windows based folder at c:\Repo\Blog\.

{% highlight bash linenos %}
cd /mnt/c/Repo/Blog
git clone https://github.com/joshgerdes/jekyll-uno
cd jekyll-uno/
sudo bundle exec jekyll serve
{% endhighlight %}

Oh, it fails

![failsoserveruno](https://content.screencast.com/users/eosfor/folders/Snagit/media/850f2eb3-0e41-4b39-a99b-38408e5b95d7/12.30.2017-21.47.png)

Ok, following the messages ```bundle install```, and it fails again

![unofailsagain](https://content.screencast.com/users/eosfor/folders/Snagit/media/4bc72398-a4ca-4c8d-811a-86540df8c3ef/12.30.2017-21.50.png)

crazy crazy! Well, few lines below we see:

![possibleissue](https://content.screencast.com/users/eosfor/folders/Snagit/media/4981c577-da96-4492-9feb-36681c4cbdcc/12.30.2017-21.53.png)

So lets try to install it. First, check if it is already there. Not sure if i'm doing this in a correct way, but... Hmm, there is something there. 

![ifzlibisinstalled](https://content.screencast.com/users/eosfor/folders/Snagit/media/0b2c19f0-a19a-40a7-8ce5-a08d1675f8f1/12.30.2017-21.57.png)

Let's also see what it is out there in repo.

![zlibintherepo](https://content.screencast.com/users/eosfor/folders/Snagit/media/4cdcf035-bb17-46c7-b004-9821c7326cbe/12.30.2017-22.02.png)

And here i'm like "WAAT?". Which one of these should i install. Oh my... This is crazy! What is the correct way for doing that? What if i have no bing/google? What do i do?

Ok, after some searching i got the following example ```sudo apt install zlib1g-dev ``` and after that ``` bundle install ``` ... And it looks better now

![bundleinstallok](https://content.screencast.com/users/eosfor/folders/Snagit/media/1edf330b-42ab-45b7-86b4-e038f286b813/12.30.2017-22.17.png)

Let's try to serve it once again ``` bundle exec jekyll serve ```, and here we got

![itworks](https://content.screencast.com/users/eosfor/folders/Snagit/media/475e5afe-7091-433a-a162-56a9a26ba20e/12.30.2017-22.20.png)

## Conclusion

What i can say here? First of all, kudos to MSFT. They made it possible to run Linux binaries natively on Windows. This simplifies every thing a lot. I'm waiting for times when we could run all types of software made for Linux directly in Windows. This going to make Linux a Windows app ;).

## Next step
Next time we going to put some modifications on this. 
- We will put some posts
- We will intergrate [sintaxhighlighter](https://github.com/syntaxhighlighter/syntaxhighlighter) into our site
