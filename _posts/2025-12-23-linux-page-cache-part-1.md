---
title: Linux Page Cache Under the Hood (Part 1) - Introduction, Files Reads
categories: [Linux, Memory Management]
tags: [linux, files, operating-systems, page-cache]
toc: true
description:: A low-level, experimental study of the Linux page cache, analyzing file read paths and page table entries to reveal how the kernel manages file-backed memory and caching.
---

## Introduction
While diving into how the Linux kernel manages memory, I encountered the **page cache** — a critical disk cache that serves as an intermediate layer between user processes and the file system (disk storage). 

This blog post presents an experimental study of the Linux page cache.

## Some Background on Page Cache

The page cache acts as an intermediate cache between disk and processes. When a process (a user-space program) requests file data, the data is **not read directly from disk every time**. Instead, the data is first loaded into RAM and then served to the process.  
This approach avoids excessive disk operations, which are time-consuming and would otherwise cause the process to block while waiting for I/O to complete.

I am planning to read from and write to files programmatically and observe how the page cache behaves. I asked AI chatbots to suggest tools for analyzing the Linux page cache, and the following tools were recommended:

1. vmtouch  
2. fincore  
3. pgcacher  
4. cachestat and cachetop  

Out of these, I experimented with **vmtouch** and **fincore**. These two tools were sufficient for my use case, as they allowed me to visualize how file data is stored and managed within the page cache.

Now, what exactly is a *page* in the page cache?

For any given process, a file must be loaded into memory before it can be accessed. However, a process only operates using **virtual memory addresses**, since Linux abstracts and hides physical memory addresses from user space. To make this work, Linux maintains a data structure that records which virtual address maps to which physical memory address.

If Linux were to map **every single byte** individually, the overhead would be enormous. For example, mapping a 1 MB file byte by byte would require maintaining around `1,000,000` mapping entries, which is clearly inefficient.

To solve this, Linux divides files into fixed-size chunks called **pages**, which are typically **4 KB** on most Linux systems. Files are loaded into memory page by page, and each page is represented by a single entry in the mapping table.  
With this approach, loading a 1 MB file requires only `1024 / 4 = 256` page entries instead of one entry per byte.

The data structure that stores information about these pages and their corresponding memory mappings is known as the **page table**.

With this background in mind, let’s move on to observing how this works in a real Linux system.

## Experiment Tools Setup

### vmtouch

`vmtouch` is a tool used to inspect page cache information for a particular file. In addition to reporting page cache statistics, `vmtouch` can also perform operations on the page cache for a given file, which we will use throughout our experiments.

#### Installation

I am capturing the installation steps directly from the terminal. Please refer to the inline comments for details on each step.

```bash
# vmtouch is available in many Linux distribution package repositories.
# I am using an Amazon Linux instance, which does not include vmtouch
# in its default repositories. Hence, I decided to build vmtouch from source.

[root@ip-172-31-30-60 ~]# git clone https://github.com/hoytech/vmtouch.git
[root@ip-172-31-30-60 ~]# cd vmtouch/

# Attempt to build vmtouch using make
[root@ip-172-31-30-60 ~]# make

# The initial make command revealed that build tools were not installed.
# Install make and the required compiler packages.
[root@ip-172-31-30-60 ~]# yum install make cc gcc

# Build the project after installing the required tools
[root@ip-172-31-30-60 ~]# make

# Install vmtouch on the system
[root@ip-172-31-30-60 ~]# make install

# Verify the installation and display supported options
[root@ip-172-31-30-60 vmtouch]# vmtouch --help
vmtouch: invalid option -- '-'

vmtouch v1.3.1 - the Virtual Memory Toucher by Doug Hoyte
Portable file system cache diagnostics and control

Usage: vmtouch [OPTIONS] ... FILES OR DIRECTORIES ...

Options:
  -t touch pages into memory
  -e evict pages from memory
  -l lock pages in physical memory with mlock(2)
  -L lock pages in physical memory with mlockall(2)
  -d daemon mode
  -m <size> max file size to touch
  -p <range> use the specified portion instead of the entire file
  -f follow symbolic links
  -F don't crawl different filesystems
  -h also count hardlinked copies
  -i <pattern> ignores files and directories that match this pattern
  -I <pattern> only process files that match this pattern
  -b <list file> get files or directories from the list file
  -0 in batch mode (-b) separate paths with NUL byte instead of newline
  -w wait until all pages are locked (only useful together with -d)
  -P <pidfile> write a pidfile (only useful together with -l or -L)
  -o <type> output in machine-friendly format. 'kv' for key=value pairs
  -v verbose
  -q quiet
```

Although the --help option reported an error, the command still printed the supported options. We will use several of these options in the upcoming experiments to understand how the Linux page cache behaves.

### fincore
`fincore` is a tool similar to vmtouch that displays page cache information for a given file. In addition to what vmtouch provides, `fincore` shows the exact pages of a file that are currently loaded into memory. This is particularly useful for understanding how file data is broken into pages and cached.

#### Installation

```bash
# Clone the linux-ftools repository, which contains the fincore tool
[root@ip-172-31-30-60 vmtouch]# git clone https://github.com/david415/linux-ftools.git
[root@ip-172-31-30-60 vmtouch]# cd linux-ftools/

# Run the configure script to generate a Makefile
# with the appropriate build options for the system
[root@ip-172-31-30-60 linux-ftools]# ./configure
checking for a BSD-compatible install... /usr/bin/install -c
checking whether build environment is sane... yes
.....
configure: creating ./config.status
config.status: creating Makefile
config.status: executing depfiles commands

# Build the tools using make
[root@ip-172-31-30-60 linux-ftools]# make

# Install fincore and related tools on the system
[root@ip-172-31-30-60 linux-ftools]# make install

# Display fincore usage and supported options
[root@ip-172-31-30-60 linux-ftools]# fincore
fincore version 1.0.0
fincore [options] files...

  --pages=false      Don't print pages
  --summarize        When comparing multiple files, print a summary report
  --only-cached      Only print stats for files that are actually in cache.
```

The available fincore options are displayed above. We will experiment with these options in the upcoming sections to observe and analyze page cache behavior in more detail

### Sample File Creation

We will create a sample file for the experiment. I will use the `dd` tool to create this file. The `dd` utility accepts an input source and an output destination and copies data from one location to another. Refer to the references section for more details on the `dd` tool.

```bash
# Input source is /dev/zero, which provides a continuous stream of '\0' (null) bytes
# Output is written to a file named "samplefile"
# A block size of 1 MB is used, and 20 blocks are written, resulting in a 20 MB file
# The file is fully populated with data, so it is NOT a sparse file
# This is important because sparse files may report a large logical size
# while consuming less physical disk space, which also affects page cache behavior
# For this experiment, we need a fully allocated file to clearly visualize page cache usage

[root@ip-172-31-30-60 pagecache-test]# dd if="/dev/zero" of="samplefile" bs=1M count=20
20+0 records in
20+0 records out
20971520 bytes (21 MB, 20 MiB) copied, 0.00770455 s, 2.7 GB/s

# Verify the file size using ls with --size
# The first "20M" indicates disk usage
# The second "20M" indicates the logical file size stored in metadata
# Since both values match, this confirms the file is not sparse

[root@ip-172-31-30-60 pagecache-test]# ls -ltrh --size samplefile
20M -rw-r--r--. 1 root root 20M Dec 23 06:01 samplefile
```
The output confirms that the file occupies the full 20 MB on disk and has a corresponding logical size of 20 MB, making it suitable for page cache experiments.

### File Read – System Calls

Linux provides several system call variations for reading from and writing to files. Among these, I will focus on the following two, as they represent the most common and important access patterns with respect to the page cache:

1. `read()`
    The `read()` system call copies data from a file into a user-provided buffer. If the requested data is not already present in the page cache, the kernel reads it from disk into the page cache and then copies it into the process’s user-space memory.
2. `mmap()`
    The `mmap()` system call maps a file directly into a process’s virtual address space. Instead of explicitly copying data into a user buffer, the file’s contents are accessed through memory loads and stores. Actual disk I/O happens on demand via page faults.

#### Difference Between `read()` and `mmap()`

Although both `read()` and `mmap()` interact with the page cache, they differ in how data is accessed by the process:

- With **`read()`**, file data is first brought into the **page cache** (if not already present). The kernel then **copies** this data from the page cache into a user-space buffer. Multiple `read()` calls may be needed, and system calls such as `lseek()` are required to change the file offset when accessing different parts of the file.
- With **`mmap()`**, the file is **mapped** into the process’s virtual address space. The initial `mmap()` call does not immediately load the file contents into memory. Instead, pages are loaded **lazily** from disk into the page cache when the process accesses them, triggered by page faults. The process can then access file data directly as a memory region (for example, as an array of bytes), without explicit read or seek operations.
- `read()` performs an explicit copy from the page cache to user memory.
- `mmap()` allows direct access to page cache-backed pages via the process’s virtual address space, with data being loaded on demand.

In the following sections, we will observe and compare how these two access methods affect page cache population and behavior in practice.

## Page Cache Behavior on the `read()` System Call

I am using a Python script to perform a `read()` system call and observe how the Linux page cache behaves. Below is the Python script:

```python
with open("/root/pagecache-test/samplefile", "br") as f:
    print(f.read(2))
```

The script opens the file using the `open()` call, then uses the returned file object to read content via the `read()` method, which internally invokes the read system call.

To understand the actual system calls made to the Linux kernel, I used strace. The relevant findings are explained with inline comments below:

```bash
# Ignore the initial system calls made by the Python interpreter to set up its environment.
# The calls we are interested in start from openat.
[root@ip-172-31-30-60 pagecache-test]# strace python3 read-system-call.py
execve("/usr/bin/python3", ["python3", "read-system-call.py"], 0x7fff4ecd83f8 /* 21 vars */) = 0
...
# The system calls for our program logic begin here.
# openat opens the file, creates a file descriptor (FD), and returns it for subsequent read/write operations.
openat(AT_FDCWD, "/root/pagecache-test/samplefile", O_RDONLY|O_CLOEXEC) = 3
# Python performs basic checks to verify that the file is readable.
fstat(3, {st_mode=S_IFREG|0644, st_size=20971520, ...}) = 0
ioctl(3, TCGETS, 0x7ffd70f552d0)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(3, 0, SEEK_CUR)                   = 0
# This is the actual read call. The FD 3 (from openat) is passed, along with a buffer.
# By default, Python requests a full page (typically 4096 bytes on x86_64) from the kernel.
read(3, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 4096
# The first 2 bytes are printed to stdout.
write(1, "b'\\x00\\x00'\n", 12b'\x00\x00'
)         = 12
# The file descriptor is closed, freeing associated resources.
close(3)                                = 0
...
+++ exited with 0 +++
```

From the trace, Python reads 4096 bytes (one page) even though the script only requested 2 bytes. This should load one page from disk into memory (page cache). Let's verify this with the `vmtouch`:

```bash
# vmtouch shows how many pages of the file are currently resident in memory.
[root@ip-172-31-30-60 pagecache-test]# vmtouch samplefile
           Files: 1
     Directories: 0
  Resident Pages: 4/5120  16K/20M  0.0781%
         Elapsed: 9.4e-05 seconds
```

Instead of the expected 1 page, 4 pages (16 KB) are loaded into the page cache.
This extra data is due to Linux's readahead feature: when a process reads a page sequentially from a file, the kernel proactively fetches additional nearby pages into the cache, anticipating future sequential reads.
To reduce readahead, we can hint to the kernel that the access pattern will be random, which typically disables or minimizes readahead. This is done using the `posix_fadvise()` system call with the `POSIX_FADV_RANDOM` flag. Note that this is only a hint—the kernel may still choose to perform some readahead.

```python
import os

with open("/root/pagecache-test/samplefile", "br") as f:
    fd = f.fileno()
    os.posix_fadvise(fd, 0, os.fstat(fd).st_size, os.POSIX_FADV_RANDOM)
    print(f.read(2))
```

Tracing this version shows the fadvise call:

```bash
[root@ip-172-31-30-60 pagecache-test]# strace python3 read-with-random-system-call.py
......
openat(AT_FDCWD, "/root/pagecache-test/samplefile", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=20971520, ...}) = 0
ioctl(3, TCGETS, 0x7fffb711e060)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(3, 0, SEEK_CUR)                   = 0
fstat(3, {st_mode=S_IFREG|0644, st_size=20971520, ...}) = 0
# The fadvise hint advising random access for the entire file.
fadvise64(3, 0, 20971520, POSIX_FADV_RANDOM) = 0
read(3, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 4096
write(1, "b'\\x00\\x00'\n", 12b'\x00\x00'
)         = 12
close(3)                                = 0
rt_sigaction(SIGINT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7ffbca83fc30}, {sa_handler=0x7ffbcacd5005, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7ffbca83fc30}, 8) = 0
munmap(0x7ffbcab1b000, 151552)          = 0
exit_group(0)                           = ?
+++ exited with 0 +++
```

Checking the page cache to check how the page cache is loaded this time with random access:

```bash
# Looks like the page cache still the same. Then found that last programm we ran already loaded the pages so this will remain the same.
[root@ip-172-31-30-60 pagecache-test]# vmtouch samplefile
           Files: 1
     Directories: 0
  Resident Pages: 4/5120  16K/20M  0.0781%
         Elapsed: 9.3e-05 seconds
# Evict all pages of the file from cache.
[root@ip-172-31-30-60 pagecache-test]# vmtouch -e samplefile
           Files: 1
     Directories: 0
   Evicted Pages: 5120 (20M)
         Elapsed: 2.3e-05 seconds
# Confirm the cache is empty.
[root@ip-172-31-30-60 pagecache-test]# vmtouch samplefile
           Files: 1
     Directories: 0
  Resident Pages: 0/5120  0/20M  0%
         Elapsed: 0.000109 seconds
# Executing the python code again.
[root@ip-172-31-30-60 pagecache-test]# strace python3 read-with-random-system-call.py
# Run the script again with the POSIX_FADV_RANDOM hint.
# Now check the cache:
[root@ip-172-31-30-60 pagecache-test]# vmtouch samplefile
           Files: 1
     Directories: 0
  Resident Pages: 1/5120  4K/20M  0.0195%
         Elapsed: 9.3e-05 seconds
```

## Page Cache Behavior on the `mmap()` System Call

Now let's examine how the `mmap()` system call affects page cache loading. We'll use a similar Python script to the one for the `read()` system call, but this time leveraging the `mmap` module (which provides access to the underlying `mmap` system call).

```python
import mmap

with open("/root/pagecache-test/samplefile", "r") as f:
    with mmap.mmap(f.fileno(), 0, prot=mmap.PROT_READ) as mm:
      print(mm[:2])
```

This script behaves similarly on the surface: it opens the file, accesses the first 2 bytes, and prints them. However, `mmap()` maps the entire file into the process's virtual address space, returning a memory-mapped object that acts like a byte array. Accessing mm[:2] triggers a page fault, causing the kernel to load the necessary page(s) from the file (via the page cache) into physical memory on demand.

Let's trace it with strace to see the underlying system calls: 

```bash 
[root@ip-172-31-30-60 pagecache-test]# strace python3 mmap-system-call.py
...
openat(AT_FDCWD, "/root/pagecache-test/samplefile", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=20971520, ...}) = 0
ioctl(3, TCGETS, 0x7fffa716e010)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(3, 0, SEEK_CUR)                   = 0
ioctl(3, TCGETS, 0x7fffa716de30)        = -1 ENOTTY (Inappropriate ioctl for device)
fstat(3, {st_mode=S_IFREG|0644, st_size=20971520, ...}) = 0
fcntl(3, F_DUPFD_CLOEXEC, 0)            = 4
# The initial calls are similar to those in the read() example.
# Here is the key mmap() call, mapping the entire 20 MiB file into memory.
# It returns a virtual address (e.g., 0x7fca4f000000) where the file content is accessible.
mmap(NULL, 20971520, PROT_READ, MAP_SHARED, 3, 0) = 0x7fca4f000000
write(1, "b'\\x00\\x00'\n", 12b'\x00\x00'
)         = 12
close(4)                                = 0
munmap(0x7fca4f000000, 20971520)        = 0
close(3)                                = 0
rt_sigaction(SIGINT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fca5dc3fc30}, {sa_handler=0x7fca5e0d5005, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fca5dc3fc30}, 8) = 0
munmap(0x7fca5df1b000, 151552)          = 0
exit_group(0)                           = ?
+++ exited with 0 +++
```

Note that accessing mm[:2] triggers a minor page fault (not visible in strace), which populates the page cache.

To isolate the effect, I evicted the page cache just before running the script. Checking with vmtouch:

```bash
[root@ip-172-31-30-60 pagecache-test]# vmtouch samplefile
           Files: 1
     Directories: 0
  Resident Pages: 32/5120  128K/20M  0.625%
         Elapsed: 9.3e-05 seconds
```

Even though only 2 bytes were accessed, the kernel loaded 32 pages (128 KB) into the page cache. This shows more aggressive readahead with mmap() compared to read() (which loaded 4 pages by default). The kernel anticipates sequential access patterns common with memory-mapped arrays and prefetches more data upfront.

we can provide access pattern hints using `madvise()` (via `mm.madvise()` in Python) instead of `posix_fadvise()`. Here's the modified script:

```python
import mmap
import time

with open("/root/pagecache-test/samplefile", "r") as f:
    with mmap.mmap(f.fileno(), 0, prot=mmap.PROT_READ) as mm:
      mm.madvise(mmap.MADV_RANDOM)
      print(mm[:2])
```

After evicting the cache again:

```bash
[root@ip-172-31-30-60 pagecache-test]# vmtouch -e samplefile
           Files: 1
     Directories: 0
   Evicted Pages: 5120 (20M)
         Elapsed: 2.1e-05 seconds
[root@ip-172-31-30-60 pagecache-test]# strace python3 mmap-with-random-system-call.py
....
mmap(NULL, 20971520, PROT_READ, MAP_SHARED, 3, 0) = 0x7f7096e00000
# The madvise() call hints random access for the mapped region.
madvise(0x7f7096e00000, 20971520, MADV_RANDOM) = 0
write(1, "b'\\x00\\x00'\n", 12b'\x00\x00'
)         = 12
close(4)                                = 0
munmap(0x7f7096e00000, 20971520)        = 0
close(3)                                = 0
rt_sigaction(SIGINT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f70a5a3fc30}, {sa_handler=0x7f70a5ed5005, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f70a5a3fc30}, 8) = 0
munmap(0x7f70a61c7000, 151552)          = 0
exit_group(0)                           = ?
+++ exited with 0 +++
# Now let's check the vmtouch output
# As expected we see 1 page of data alone loaded into the memory 
[root@ip-172-31-30-60 pagecache-test]# vmtouch samplefile
           Files: 1
     Directories: 0
  Resident Pages: 1/5120  4K/20M  0.0195%
         Elapsed: 9.3e-05 seconds
```

## Page Cache Behavior on the `open()` System Call

Having observed page cache loading during `read()` and `mmap()` system calls, I was curious whether the `open()` system call alone causes any data to be loaded into the page cache for a particular file. Let's test this:

```python
with open("/root/pagecache-test/samplefile", "r") as f:
    print("File Opened Complete")
```

This script simply opens the file and prints a message—it performs no actual read operations. After evicting the page cache beforehand and running the script, I checked the cache status with `vmtouch`:

```bash
[root@ip-172-31-30-60 pagecache-test]# vmtouch samplefile
           Files: 1
     Directories: 0
  Resident Pages: 0/5120  0/20M  0%
         Elapsed: 9.6e-05 seconds
```

As shown, no pages are resident in the cache. This confirms that merely opening a file does not populate the page cache. Data is loaded only when the file is actually accessed (e.g., via read(), memory-mapped access triggering a page fault, or other operations that demand file content).

## Conclusion

By now, we've gained a solid introduction to the Linux page cache through practical experiments with `read()`, `mmap()`, and `open()` system calls. We've seen how the kernel eagerly populates the cache with readahead for sequential patterns and how hints like `POSIX_FADV_RANDOM` or `MADV_RANDOM` can minimize unnecessary prefetching.

This naturally leads to deeper questions: What happens during truly **random access** patterns? How are file pages loaded on demand (e.g., via page faults in `mmap()`)? What do the process's **page table entries** look like for file-backed memory, and how does the kernel maintain them efficiently.

In the next part of this series, we'll dive into these topics with more experiments.

## References

This series of experiments was inspired by and builds upon the excellent blog post by Viacheslav Biriukov:

- **Linux Page Cache for SRE** – A deep dive into the Linux page cache.  
  [https://biriukov.dev/docs/page-cache/0-linux-page-cache-for-sre/](https://biriukov.dev/docs/page-cache/0-linux-page-cache-for-sre/)

### Tools and Documentation

- **vmtouch** – Virtual Memory Toucher, used to inspect and control the file system cache.  
  Official page: [https://hoytech.com/vmtouch/](https://hoytech.com/vmtouch/)  
  Source code: [https://github.com/hoytech/vmtouch](https://github.com/hoytech/vmtouch)

- **fincore** – A utility to count the number of cached pages for files (alternative to vmtouch).  
  Source code: [https://github.com/david415/linux-ftools](https://github.com/david415/linux-ftools)

- **Python mmap documentation** – Detailed explanation of memory-mapping files in Python.  
  [Python mmap: Improved File I/O With Memory Mapping – Real Python](https://realpython.com/python-mmap/)

Additional useful kernel documentation:

- **posix_fadvise(2)** – Linux man page for file advice.  
  [https://man7.org/linux/man-pages/man2/posix_fadvise.2.html](https://man7.org/linux/man-pages/man2/posix_fadvise.2.html)

- **madvise(2)** – Linux man page for memory advice.  
  [https://man7.org/linux/man-pages/man2/madvise.2.html](https://man7.org/linux/man-pages/man2/madvise.2.html)