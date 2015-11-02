---
layout: post
title: The meaning of top command
category: [tech]
tags: [linux]
---

Here is a snapshot of top command:

![top]({{ site.baseurl }}/images/top.png).

And I searched the meaning of the items:

## Cpu(s)
- us: the percentage of time spent on user process
- sy: the percentage of time spent on system call
- ni: the percnetage of time spent on user process that is **niced**
- id: idle time
- wa: cup time spent on waiting for io
- hi, si: interrupt
- st: time waiting for hypervisor to serve another virtual cpu (for virtual matchine)

## load average
The three number means the load average of 1, 5, 15 minutes.

On single core system, 1.00 means cpu is fully utilized, while 0.00 means totally idel, number greater 1.00 means trouble.

On multicore system, number of cores means fully utilized.

For this server, it has 32 * 8 = 256 cores, so the 100% load will be 256?
