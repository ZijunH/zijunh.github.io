---
layout: post
title: "aria2 assisted downloading in China"
date: 2019-12-18 17:31:24.000000000 +8:00
excerpt: Using aria2 to increase downloading speeds in China.
tags: 
  - discussion
keywords: [aria2, downloading, china, discussion, fast]
category: random
---

Those who have been to China are often frustrated by the restrictions imposed by the Great Fire Wall of China (GFW). Even though VPNs are available to bypass the restrictions, they are often very slow or unstable. However, even for foreign websites that are not blocked by the GFW, accessing them is extremely slow. This becomes extremely problematic when you need to download large files, such as software updates or other essential files. The slow speed of software updates can be remedied by changing mirror sites to those local to China. However, the file I needed was only hosted on 1 website. The speeds were painstakingly slow (often ~90kb/s), it would take hours to download a 1GB file, even when I have a bandwidth of 100MBps.

To solve this problem, the idea of parallel downloading can be introduced. [aria2](https://github.com/aria2/aria2) is such a tool. It allows connection to single or multiple hosts from the same machine. This allows multiple connections to the same host to increase the download speed. Each connection downloads a chunk, and when a chunk is downloaded, a new connection is made to download the next chunk. "-s" specifies the number of connections to the hosts, "-k" specifies the size of each chunk, and "-x" specifies the maximum number of connections. The full documentation is [here](https://aria2.github.io/manual/en/html/).

The arguments I used were:

```
aria2c.exe -x16 -s16 -k1M link
```

Note this method requires manually downloading the file. Perhaps manual interference can be omitted by writing a local proxy that automatically does this. However, I am not a network expert so this will be a task for the readers (or when I do become a network expert).