---
layout: post
title: Development Enviroment with Vagrant
category: [tech]
tags: [vagrant]
---
# Goal #
Migerate my development enrioment to vagrant managed virtual box.

# vagrant #

## nfs ##
It is said that common shared folder will be very slow. I decided to use NFS although I didn't measure the difference myself...

I followed the instructions [here](https://docs.vagrantup.com/v2/synced-folders/nfs.html).

Put it simply, we need to do two things: set private networks, set share folder as nfs. Configure file is as belows:
{% highlight ruby %}
# set network as dhcp
config.vm.network "private_network", type: "dhcp"
# set shared folder, and make type nfs
config.vm.synced_folder "../Develop/", "/develop", type: "nfs"
{% endhighlight %}
The bad thing is that, after nfs is set, we will be prompted with password everytime. Vagrant document gives the way to avoid this.

## port forwarding ##
{% highlight ruby %}
config.vm.network "forwarded_port", guest: 8080, host: 1234
config.vm.network "forwarded_port", guest: 8081, host: 1235
{% endhighlight %}
The good part of vagrant is that, when port conficts it will explictly complains which is far better than virutal box.

## provisioning ##
Have't investigate this yet..

# struggle for install #

## install recent repository ##
use this command:
    rpm -Uvh http://mirror.webtatic.com/yum/el6/latest.rpm

## update php version from 5.3 to 5.4 ##
somehow the default php version in CentOS6 is 5.3, which doesn't accomendate my project. So I need to update php version with:

    yum install yum-plugin-replace
    yum replace php-common --replace-with=php54w-common

As a result, this step brings lots of unexpected problems.

## install gcc, apache, mysql ##

## install redis and memcached ##
    yum install redis memcached
    yum install php54w-memcache (not php-memcache!)

when I tried to install php54w-redis, I found out that there was no such package. So I had to install it from source code phpredis...

## start my project ##
I encountered following problems:

* white page: My project is based on CodeIgniter. The werid part of the framework is that, its hides all the error message when doing mysql connect with @.
  As a result, it took me lots of time to find out that reason is simply mysql server didn't started.
* no error log: it turns out the apache didn't have the write permission to the log folder
* 