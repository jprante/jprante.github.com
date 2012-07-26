---
layout: post
category: applications
title: "Memory-mapped files with Lucene: some more aspects"
tagline: "by Jörg Prante"
tags: [lucene, elasticsearch, unix, linux, solaris, virtual memory]
comments: true
---

![Server Mainboard](/assets/images/flickr-2142582850-original.jpeg)

*[Licensed under Creative Commons](http://creativecommons.org/licenses/by/2.0/)*

In a great article about using memory-mapped files in Lucene, Uwe Schindler discussed [why switching Lucene directories to MMapDirectory](http://blog.thetaphi.de/2012/07/use-lucenes-mmapdirectory-on-64bit.html) is most preferable today.

Some questions have arisen in the article and some thoughts came up in my mind, so I decided to write a little more about virtual memory.

One question that came up to me was: why are memory-mapped files a better choice over paging - since both of them are managed by the OS, and both try to optimize memory resource usage between memory and external disks?

Or: is every 64bit system suitable for memory-mapped files?

Another one: do Linux/Solaris offer adequate solutions to the challenges of switching to ``mmap``'ed Lucene directories?

And finally: are there already answers to ``mmap``-related questions?

Before stepping into trying to find answers, let me quickly resume what the paging feature is as we know it from UNIX-like operating systems.

Paging
------

Paging takes place on the operating system (OS) level. The OS has to manage process code, data, and several types of cache, for instance the file system cache, so they can exist side-by-side in the virtual memory address space. Because the OS knows about what is going on in the whole system, it can take care of the right priorities. Memory pages can be moved to RAM or they can be evicted to external storage. Applications in the user space can not force the OS to change paging strategies. Paged data can be adressed only by the address of the page. When memory pages are swapped, they usually can not be shared between processes.

Swapping or (paging) is generally known as a performance killer because it is known to cause high I/O activity on swap devices or files on disk. But paging is not evil. It is very useful for many cases. Long lived application and large amounts of rarely demanded memory can be swapped to disk. By doing this, the OS is able to fight back against bad inefficient programming style and bad behaving processes. In such cases, paging works like a magic cache, just delivering the right memory resources at the right time to the right place.

But the page cache not a usual cache. It is a high-priority kernel task that manages paging. And it also depends on the operating system how paging works. Solaris, Linux, Windows, Mac OS X have all different paging behavior. 

Why is disabling paging useful? There are two kinds of applications to limit paging: high-security (to keep secrets like encryption keys in memory) and real-time applications to improve low latency.

Paging can be disabled by super user privileges. But when paging is disabled, the memory pressure gets higher and the OS has lower file system cache available because it must compete with all the process code and data more frequently. It depends on your OS if preventing paging is desirable, but one fact is, your kernel will spend more time in overhead routines for managing memory allocation.

On Linux, for example, you could put a lot of RAM into your machine to ensure the amount of RAM is greater than your applications will ever need and you can disable swap. If you want to disable swap, terminate all processes that use swap, and execute the command (as root):

	# swapoff -a

On Solaris, you should be aware that ``/tmp`` is located in the swap area and writing to ``/tmp`` may conflict with the idea of completely disabling paging. Additional issues may arise when using ZFS. ZFS uses an ARC (adaptive replacement cache) but not the normal page cache. The ARC lives in the kernel space. By default, ARC is allocating all available memory for aggressive caching, where swap space is managed by ZFS devices. The result is extra memory pressure when applications with large heap requirements are present, like Lucene. To remedy the situation, it is often recommended to reduce the ARC size so that ARC plus other memory use fits into total RAM size. Use at least Solaris 10 10/09, and for example, you can limit ARC cache usage with adding a line such 

	set zfs:zfs_arc_max 0x780000000

to /etc/system. While ``mmap()`` usually accesses the page cache and not ARC, it is possible that the OS may need to keep two copies of the data around. The latest ZFS implementations try to avoid such issues.

Memory footprint of a JVM process
---------------------------------

For understanding better where the JVM puts memory-mapped files, let's examine the memory organization of a JVM process. Mainly it consists of:

- the heap. The maximum size is given by the flag ``-Xmx``. This is where Java objects are allocated.
- the permanent generation heap ("perm gen"). The maximum size is given by the flag ``-XX:MaxPermSize``. This is where the class metadata objects, interned strings (String.intern), and symbol data are allocated.
- the code cache. The JIT compiled native code is allocated here.
- memory mapped .jar and .so files. The JDK's standard class library and application's jar files are often memory mapped. Various shared library  and application shared library files are also memory mapped.
- thread stacks. The maximum size is given by flag ``-Xss`` or ``-X:ThreadStackSize``.
- the C/malloc heap. Both the JVM itself and any native code typically uses ``malloc`` to allocate memory from this heap. NIO direct buffers are allocated via malloc.
- any other ``mmap`` calls. Native code can allocate pages in the address space using ``mmap`` (or Java code via JNA)

Most of these allocations are allocated in terms of virtual memory early, but committed only on demand. Your application's physical memory use may look small at start time, but may get higher later on.

We learned that memory-mapped files in Lucene allocate pages outside of the Java and C code heap spaces, instead it allocates them directly in the system's virtual address space.

Memory-mapping
--------------

A memory-mapped file is virtual memory which has been assigned a direct byte-for-byte correlation with some portion of a resource. This resource is typically a file that is physically present on-disk, but can also be a device, shared memory object, or other resource that the OS can reference through a file descriptor.

Memory-mapped files can be shared between processes (this might be complex though). When Java directly allocates buffers or maps files to memory, they are allocated outside the Java heap.

Why ``mmap`` is getting more important today
--------------------------------------------

One important argument for ``mmap`` is the I/O performance. Much progress has been made in input/output file processing. In recent years, new server machines in the PC class (Intel CPU architecture) were equipped with more powerful memory management but also with new I/O devices. 

The challenge was to cope with huge amounts of RAM (because RAM chips were getting cheaper and denser) and with overhauled virtual machine designs where the total address space is virtualized to several logical units that look like a complete hardware layer to the OS. As a result, the memory resources in hardware today can be efficiently managed better than ever. 

Another trend in overall system performance is that compiler tool chains and virtual machine architectures (like the JVM) got tremendously better optimization techniques for local code and local data, avoiding much of cache misses and page faults that lead to paging. For applications with large heaps like Lucene, avoiding unnecessary paging is important. It can slow down the overall performance of a machine significantly.

64bit hardware - it evolved over time
--------------------------------------

The largest memory address you can point to with a 32bit pointer is 4GB. But is it really enough to know that 64bit operating systems are using 64bit address pointers? The answer is, it depends. Some engineering decisions for the hardware of our PC "industry standard" server a few years ago regarding memory subsystems were suboptimal, mostly because it was too expensive for the vendors to build such machines. UNIX servers kept for some time their advantage in virtual memory management over PC server. But today, the situation has much improved in favor of the PC servers.

Yesterday's 64bit hardware was

- limited by RAM slots, maximum capacity of 4 or 8 GB

- not able to push large gigabytes of data between CPU and memory subsystems at high speed

- MMU address space: 43bit (PowerMac, SUN UltraSPARC III) = 8TB, 48bit (AMD64) = 256 TB

- limited by address bus width: most manufacturers did not exploit the address space beyond 4 GB because it was too expensive. For Intel PCs in the pre-AMD64 era, PAE (page address extension) allowed for addressing up to 64 GB

- and it was limited when reading or writing to external storage devices (by SCSI speed and drives)

whereas today's 64bit hardware

- have lots of memory banks in machines, with more than terabyte capacity

- have blazingly fast buses for transports between CPU and main memory subsystems

- have MMU integrated into the CPU (HyperTransport, QuickPath Interconnect) with serial data paths that can connect to anything on the main board

- and last not least they have very fast read and write speeds on external storage devices with a improved bus architecture and buffer hierarchy (SAS, SSD)

And fortunately, beside the application code, the operating systems were improved.

So when you are able to throw in as much hardware as you can buy into your system and the OS can scale with it, you will not need bothering too much about the consequences when you enable ``mmap()``'ed files.

Memory overcommit
-----------------

Memory overcommit is a nifty kernel feature of Linux/BSD/AIX where ``malloc`` never fails. It never returns a NULL pointer. In Linux, this is usually enabled by default. Processes are allowed to allocate more virtual memory than the system actually has, on the hope that they won't end up using it. If processes try to use more memory than is available, the Out-of-Memory killer (OOM killer) comes in and picks some process to exit it immediateley in order to recover memory for the operating system.

Solaris has no memory overcommit feature.

This feature is also relevant for Lucene ``mmap``'ed processes. When they request memory, it is likely they will not fail. But later on, when the process memory is going to be used, the OOM killer might step in and kill the Lucene process.

Locking Lucene processes to memory with ``mlockall``
----------------------------------------------

Memory overcommit does not tell if a Lucene process completely resides in RAM for low latency. One approach to enure low latency is locking the pages of the Lucene process to RAM.

``mlockall`` is a UNIX system call that causes all of the pages mapped by the address space of a process to be memory resident until unlocked or until the process exits or execs another process. 

Basically it works like removing paging for this process from your system. With an ``mlockall``'ed Lucene, you can be sure that your Lucene process will always reside in RAM. When you start a process, call ``mlockall``, and it won't fit into RAM, you will get an error, although memory overcommit is enabled.

``mlockall`` and ``swapoff -a`` have in common that both prevent paging, but one with sharp precision, and the other one like a hammer to the head.

As Linux kernel hacker Robert Love pointed out: 

> "Using swapoff for either of the aforementioned use cases is suboptimal. Even if paging is detrimental to a specific application, it is beneficial to the system as a whole. Thus disabling an entire system's paging isn't a best practice. If your application holds secrets in memory that you don't want paged to disk or if your application requires deterministic timing, consider using mlock() or mlockall() to prevent paging."

So finally, we have found out why we should prefer ``mlockall`` and not disabling the entire system's paging.


Configuring ``mmap`` and ``mlockall`` in Elasticsearch
-------------------------------------------------------

Elasticsearch is my favorite distributed search and indexing implementation based on Lucene.

Example configuration for enabling ``mmap``'ed Lucene directories in Elasticsearch indexes in elasticsearch.conf: 

	index: 
	  store: 
	    type: mmapfs 
	    fs: 
	      mmapfs: 
	        enabled: true

Elasticsearch allows an ``mlockall()`` call at node booting time with the parameter 

	bootstrap.mlockall: true

in elasticsearch.yml. To complete the configuration the JVM memory size should also be configured. This is done by editing ``ES_HEAP_SIZE`` in the start script.

``bootstrap.mlockall`` might cause the JVM or shell session to exit if allocation of the memory fail with ``unknown mlockall error 0``. Possible reason is that not enough lock memory resources are available on the machine. This can be checked by ``ulimit -l``. The value should be set to ``unlimited``.

A final word
------------

I traced the 64-bit server trends now for more than ten years. The future is not hard to see. Servers will be equipped with more and faster RAM, and terabytes of main memory will become popular. For Lucene-based applications it means that RAM-only installations will become more important.

Uwe Schindler's article reminded me of the huge progress of recent server technology. I appreciate all the hard work of engineering such extraordinary systems that most annoying resource limits and performance bottlenecks almost have become part of the past.

If you belong to those who are not blessed with the latest and greatest 64bit PC server hardware available or you are simply tied to maintain older OSs and applications, do not expect too much from ``mmap``'ed Lucene directories. With older hardware, obsolete operating system versions, or tight memory resources, ``mmap``'ed Lucene directories and ``mlockall``ed Lucene processes are not always a neat solution.

References
----------

Red Hat Enterprise Linux 3: System Administration Guide, "Removing swap space" -  <http://docs.redhat.com/docs/en-US/Red_Hat_Enterprise_Linux/3/html/System_Administration_Guide/s1-swap-removing.html>

Ulrich Gräf, "Performance mit ZFS - mit Datenbank Tipps" (in german) - <http://www.as-systeme.de/sites/default/files/events/zfs_1004.pdf>

Hiroshi Yamauchi, "JVM process memory" - <http://hiroshiyamauchi.blogspot.de/2009/12/jvm-process-memory.html>

Linus Torvalds, "How big is a 64 bit address space?" - <http://www.realworldtech.com/forum/?threadid=30389&curpostid=30406>

Norm Murray and Neil Horman, "Understanding Virtual Memory" - <http://www.redhat.com/magazine/001nov04/features/vm/>

Randy Bryant and Dave O’Hallaron, "Virtual Memory: Systems" - <http://www.cs.cmu.edu/afs/cs/academic/class/15213-f10/www/lectures/16-vm-systems.pdf>

University of Alberta, "Understanding Memory" - <http://www.ualberta.ca/CNS/RESEARCH/LinuxClusters/mem.html>

"The Solaris Memory System - Sizing,Tools and Architecture" - <http://www.solarisinternals.com/si/tools/memtool/vmsizing.pdf>

"Respite from the OOM killer" - <http://lwn.net/Articles/104179/>

"What are the main differences between mlockall() and swapoff -a?" - <http://www.quora.com/What-are-the-main-differences-between-mlockall-and-swapoff-a>

"Elasticsearch guide: Installation" - <http://www.elasticsearch.org/guide/reference/setup/installation.html>

