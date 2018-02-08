---
layout: post
title: Writing a Memory Allocator
---

Memory allocation is a vast and complex topic and this post by no means
tries to be a comprehensive guide. This is an introduction to memory allocation
and how **you** can play around and write your own little memory allocator.

Memory allocation 101
---------------------

A program can dynamically allocate memory by increasing the size of the _heap_ 
(`malloc(3)` family of functions) or of the current _stack frame_ (using 
the `alloca(3)` function). This post is about allocating memory on the heap, 
which is the most common case.

Before getting our hands dirty with the memory allocation code, let's first 
have a basic understanding of the **memory layout** of a process; i.e., how a 
process is divided after being loaded in memory. The following image roughly 
illustrates the memory layout of a process on Linux.

![Memory Layout of a process in Linux]({{ site.url }}/images/process_memory_layout.png)

As we can see from the previous image, a process is divided into a couple of
_segments_. The purpose of each segment is beyond our scope right now,
so if you want to have an idea of each of them, do 
[some reading](http://en.wikipedia.org/wiki/Data_segment) and then
come back. The _size(1)_ utility allows you to see the size of each segment:

{% highlight sh %}
    $ size `which ruby`
   text	   data	    bss	    dec	    hex	filename
   2369	    672	     16	   3057	    bf1	/home/renato/.rvm/rubies/ruby-1.9.3-p194/bin/ruby
{% endhighlight %}

Looking back at the memory layout of a process, we see two important blue arrows.
The first one is the _stack pointer_ and it indicates the current top of the stack.
In other words, the frame of the function currently being executed. The second
one is called the ** _program break_ ** and it points to the current limit of
the heap. 

At program start, the program break points to the end of the bss section and if
we need more memory along the way, the heap must be resized. To do this, we need
to tell the kernel to adjust the program break so that we have some free memory 
available for our process. Important to note is that the Linux kernel will not
actually allocate physical pages when the program break is changed; they are
only allocated when the process tries to access them. This is called _memory
overcommit_ and can lead to 
[unexpected behavior](http://linuxdevcenter.com/pub/a/linux/2006/11/30/linux-out-of-memory.html).
When deallocating memory, the process is basically the same: the program break
is changed, indicanting that the process doesn't need some of the memory
it allocated before.

Changing the program break
--------------------------

On Linux, you can use the _brk(2)_ system call to directly change the program
break. It was introduced in SUSv1, marked as LEGACY in SUSv2 and removed in
SUSv3 in favor of the _malloc_ library function. For that reason, it is only
available via feature test macros.

glibc also provides a _sbrk(3)_ function built on top on _brk(2)_, which allows
the programmer to specify just the number of bytes to be incremented instead of
the actual address of the program break as in the _brk(2)_ system call.

Writing a memory allocator
--------------------------

Finally! The moment that everyone was waiting for so anxiously!

Unfortunately, it is my duty to warn the avid reader: the steps I'm going to
guide you through will build a very simple implementation of a memory allocator.
We won't be dealing with issues like thread-safety, fragmentation and many
other tweaks used to improve efficiency. 
[Papers](http://dl.acm.org/citation.cfm?id=996848) have been written on the topic.
However, you will still have a good idea of how to implement a memory allocator,
so don't go away just yet.

**The allocation function**

The signature of an allocation function such as _malloc(3)_ is:

{% highlight c %}
    void *malloc(size_t size);
{% endhighlight %}

The allocation function receives a _size_ parameter and returns a block of
free memory of _size_ bytes, or _NULL_ on error.

Here is how our allocation function is going to behave: when it is first called,
we are going to increment the program break a number of times the requested size,
so that we don't have to change it again in future calls. Remember that system
calls impose an overhead, and mantaining some free memory avoids unnecessary
calls to _brk(2)_.

When the _free_ function is called to deallocate some memory, the passed free
block is added to the extra memory previously allocated, building a _linked
list of free blocks_. Let's make it doubly linked to ease navigation
between the blocks of memory. This way, our allocation function must
mantain a free memory list similar to the one in the image below:

<img src ="/images/free_block_list.png"
alt="Linked list of free memory blocks" align="center" width="800" height="170"
title="Linked list of free memory blocks" class="img"</img>

Therefore, when the allocation function is called, all we need to do is to find
a block of free memory that is big enough for our request. If there is no
big enough block of memory, then we need to ask the operating system to
increment the program break once again. Flawless strategy!

However, one question remains unanswered: how do we know the size of each block
of memory of our free list? How do we know the size of a block of memory
passed to _free_? To work around these problems, we are going to use a neat
little trick: when allocating memory, we are going to write the size of the
block of memory in its first bytes. This way, what we are actually returning
from a call to our allocation function is something like:

<img src ="/images/allocated_block.png"
alt="Block of memory returned by our allocation function" align="center" width="600" height="210"
title="Block of memory returned by our allocation function" class="img"</img>

Voilà! Now we can iterate over our free list and inspect the size of each block
of memory to find a big enough block for a specific call to our allocation
function.

**Deallocation**

How about deallocation? In our case, it is going to be very simple: just add
the passed block of memory to the free list. You can also decrement your program
break if you have a big enough block of free memory before the it.

Obviously, you can pass the deallocation function only blocks of memory previously
allocated by the correspondent allocation function, but that is necessary with
any _malloc_ package.

Show me some codes!!
--------------------

[There](http://www.canonware.com/jemalloc/) [are](http://www.hoard.org/)
[many](http://www.malloc.de/en/) [memory](http://labs.omniti.com/labs/portableumem)
allocators out there. However, they are some orders of magnitude more complex than
the one we described here. Take a look at the Doug Lea's text on _dlmalloc_:
[http://g.oswego.edu/dl/html/malloc.html](http://g.oswego.edu/dl/html/malloc.html).
You can also peek at the source code, which is very well documented:
[ftp://g.oswego.edu/pub/misc/malloc.c](ftp://g.oswego.edu/pub/misc/malloc.c).

[I wrote an extremely simple memory allocator](https://github.com/rmascarenhas/lpi/blob/master/chap07/malloc.c)
following the guidelines described in this post. It still needs some love
but it might be good to give you a start.

Why bother?
-----------

Why bother writing a memory allocator? Mainly because you learn a lot about
memory management and start to realize how much is done under the hood when
you distractedly call _malloc_. And knowing how stuff works is always a Good Thing®.

I want moar!
------------

Great to hear that!

First, be sure to read through the man pages of the _malloc(3)_ function
and _brk(2)_ system call. [Understanding Memory](http://www.ualberta.ca/CNS/RESEARCH/LinuxClusters/mem.html)
is also a good read. Finally, I can't recommend enough
[The Linux Programming Interface](http://man7.org/tlpi/) book, by Michael Kerrisk. Most
of the content of this post was based on this book, and it is **just awesome**. Really.
Just go and get it.

Conclusion
----------

I hope you enjoyed this post and learnt something, maybe. If you have any
corrections, I'd be very glad to hear you and fix my mistakes. My main point here,
anyway, is to motivate you to try and experiment with stuff you learn: in this case,
a memory allocation strategy. Reinvent the wheel! That's how you learn: practicing.

See you next time!
