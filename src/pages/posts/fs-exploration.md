---
layout: ../../layouts/MarkdownPostLayout.astro
title: 'Basic Filesystems'
author: 'Todd'
pubDate: 2022-09-26
---

# Introduction

Recently I have decided to do a little bit of reading on filesystems. Some as a refresher and some to learning of truly new things. I remembered some of my content from my operating systems class, but that was awhile ago and most of the details I forgot. My university, for the Software Engineering program, did also not have an indepth of an Operating Systems class as others may have had. Instead of building an operating system, we focused on high level ideas around the design and implementations. Some Universities have student implement a small operating system, for my class, we built some simulations to see how various algorithms worked. As an example, for filesystems, we talked a little bit about inodes and how basic file systems are laid out, but we spent a majority of our time, and our project for that sections, on simulating an algorithm for seeks on a file system.

There is great important in understanding some of those concepts. One thing, if someone notices, of important to notice is that in many places in computing we see similarities. We often see structures and algorithms, or the ideas behind them, shared across different problem domains. An example this isn't related to filesystems per-se is with caching. At the high level, the cahce that speeds up a web server draws from the same fundamental structures and algorithms that hardware caches in a CPU draws from. I guess it does have to deal with filesystems since we often in modern computers have cached also. We cache files that we hope the CPU or user will need repeatingly to speed up our processing or to prevent from hitting slower memory regions.

While this information was still vastly helpful, I still would like to try my hand at implementing a filesystem. Afterall, the only way to say you know how a filesystem works completely is to be able to implement one. 

# Setting lofty goals

A lofty goal, since that is how I like to start. I really do like punishing myself, I often do this. A lofty goal of mine to say, "I mastered filesystems" would be to taken up a large project. So, my lofty goal would be to implement the ext4 filesystem for NetBSD. I may never actually do this, but having this goal in sight will help motive me to learn and where I should focus. With that being stated, reading a text book that gives high level details isn't sufficient. I will need to, I want to get anywere close, understand the actual algorithms and problems that present themselves. 

# Refreshing

So, before I get knee deep in the weed, I need a starting place, which is refreshing myself. For this I have cracked open my copy of *"Modern Operating Systems"* By **Andrew Tanenbaum**. While the copy that I have is the third edition, the information in it is still plenty valid since many concepts in filesystems and operating systems have remained the same or similar with just improvements overtime. 

## inode

One of the most fundamental structures of a file system is the *inode*. This contains various information about a file. The term *inode* is short for **index node**. This is what we would call the *metadata* of a file. Some examples of fields would be time stamps (last modified), block cmaps, size, filesystem version, checksum, and more. The ext4fs is fairly large. The *inode* for a file also tends to be fixed size. 

## superblock

The *superblock* is like an *inode* for the filesystem as a whole. *Superblocks* are the *metadata* to the filesystem. This will often include a magic number to be able to identify the filesystem. If you are unfamiliar with the term *magic number*, here is an example you may be familiar with. Imagine you are opening an image. Well, how does your computer know it is an image? Better yet, how does your computer even know what kind of image it is? Afterall, there are tons of formats for images; jpeg, bmp, ppm, tiff, etc. As per the Portable Pixmap Format (PPM) standards, if there is a P3 located at a certain offset of the file, then we can be sure it is a ppm file. For PPM, this will be the first two bytes of the file. Another example of an image if the bmp format. If there is an ascii "BM" in the first two bytes, then we can safely assume it is a bmp file if we are looking at images. 

# Structure for the simplest file system

The simplest way to think of a file system is continguous boxes we can place things in. To keep things simple, we can imagine all the boxes having a fixed size. Every block of the filesystem we want to be 1 kilobyte (1024 bytes). In our simple filesystem, we can imagine there are 20 boxes laid out side by side. This will come in handy in the next section(s). But for now, lets just imagine that outside of the superblock, we have 20 boxes each 1 KB "wide."

# Operations

There are a few operations a filesystem must be able to do in order to be useful. These are fairly simple and straight forward if we want to keep it basic. A filesystem must be able to read and write.

## Write

First thing we want a filesystem to be able to do is to write. Afterall, we have filesystems to keep information around for longer than lifecycle of the machine. The lifecycle being between power on and power off. We use filesystems for this long term data store. So, the first thing we should be able to do is write information to file system. We do this by filling in one of those 20 blocks above. If we have 2048 bytes of information, we can do some quick math to see how many blocks we need to use. 2048 / 1024 = 2. If this is the first file written, we will write them in box[0] and box[1] (we are programmers, so like everything zero-indexed). Now we have a file we want to write that is 800 bytes. So we write this information into box[2]. Now, we want to write another file that is 500 bytes, so we write that into box[3]. All of the "free space" that exists in block 2 is reserved, so we have to use the next entirely free box. There are some problems that will arise here, but we will leave that for future Todd. As for writing the files, we would just write the bytes sequentially to the disk.

## Read

Now that information is written, we want to be able to read. Afterall, what good is writing information to a disk if we can't read it? Otherwise, we might as well just write it to `/dev/null.` This is where those *inodes* come into play. A basic *inode* for a basic filesystem will contain two fields of importance, a starting offset and the number of blocks. Taking our above writes into consideration, if the first file we wrote is kitties.txt, we would go to the kitties.txt *inode* and see that is has an offset of 0. It is at the beginning of the filsystem. We would see that is also has 2 blocks, so we know box[0:1] are this file. In order to get the 2nd file, we would see in the *inode* that it's starting offset is 2, with a number of blocks as 1. If we wrote a new file with a size of 3000 bytes, we would know that we need 3 blocks and they would be written in box[4:6]. To retrieve this file, we would see in the *inode* that the starting offset is 4 with a number of 3 blocks.

## Delete

Lastly, a function that every filesystem should have implemented to call itself a filesystem is a delete function. For this, we simple free up the blocks to be used again. If we are deleting the first file, we simple removed the information from box[0:1] and mark it ready for use. 

## conclusion for operations

These are the two most important functions of an operating system to implement. Of course many more exist, but for brevity and just initial exploration, these are two we will focus on initially.

# Problems for future Todd

As mentioned, there is a problem we left for future Todd in the implementation section to be solved.

## Fragmentation

The first problem is fragmentation. Suppose we delete the 2nd file that takes up spaces box[2]. Box[2] is not free for use. Now imagine the rest of the filesystem is taken up, or most of it. Let us say all of the boxes from box[0:1] and box[3:18]. So we now have boxes 2 and 19 open to write. Suppose we want to write a new file to the system with 1618 bytes. So it needs two boxes. We no longer have contiguous holes, so we have two possible solutions here. The first is to defragment our filesystem by rewriting data such that we now have two contiguous boxes. We can do this by rewriting all the data in box[3:18] to box[2:17]. The trade off is, this can be an expensive operation. Before we can write, we need to read in all the information in box[3:18] and rewrite the data into box[2:17]. Then we can finally write the two blocks work of data into box[18:19].

The other possible solution is to throw out the requirement for contiguous blocks. We can change the structure of the operating system were we keep a map or a tree like structure to keep tabs on which boxes belong to which file. In our *inode* we can have a reference pointed to box[2] for the first block in the file then another reference to box[19]. So when we go to read, we know to start reading at box[2] and to also read from box[19].

## Trade offs with fragmentation

So we have saw two different possible solutions for an extremely crude filesystem. The filesystem can either group things together to keep blocks contiguous or it can use some structure that allows for any block to be used, however there are some tradeoffs to consider.

Now before going to deep, there needs to be a realization here. Some of this tradeoff goes away with newer memory systems like SSD or NVME drives. But looking at conventional HDDs, spinning rust, or old tape system, this can be a huge issue that needs to be carefully considered. On the more analog systems, reading and seeking are expensive operations. If you want the fastest reads, we want to minimize the seeks. This is where keeping blocks contigous is really helpful. Reading from box[1:2] is always faster than reading box[2] then box[19]. You are picking up latency just from the mechanical parts moving. Now with a small file system with 20 blocks at 1 KB each, this isn't too noticable. However consider a 500 GB hard drive and having to read a block somewhere to end of that. Imagine that you have to seek sequentially across the the disk. There is another aspect that comes into play. Even if you have to jump around, on many older systems, it is faster to read blocks that are ahead by many than to turn around and read the previous block. That is become some older systems could only linearly seek in one direction. So to look at box[10] then box[9], you had to seek all the way to box[19] before being able to circle back to box[9].

# Next steps

Having refreshed on some basic concepts, I think my next steps will be to look into implementing a basic filesystem. Perhaps something like above where there are only 3 functions. Before implementing something as low level in the kernel, I will take a look at Fuse which will allow me to write a userspace driver. I have also never messed with Fuse, so it will be learning some basic implementation and on top of that, learning Fuse. 
