---
title: Linux Page Cache Under the Hood (Part 2) - Page Table, Page Faults
categories: [Linux, Memory Management]
tags: [linux, files, operating-systems, page-cache]
toc: true
description:: A low-level, experimental study of the Linux page cache, analyzing file read paths and page table entries to reveal how the kernel manages file-backed memory and caching.
---

## Introduction

In the previous part, we explored the basics of the Linux page cache, analyzing how it gets populated during file reads (via `read()` and `mmap()`) and using tools to observe caching behavior. In this section, we'll perform experiments with random reads, examine how page tables work, and learn about page faults in depth.

The flow of this post follows my own journey of discovery as I experimented to understand page faults and page tables.

## Some Background on Page Faults and Page Tables

### Page Faults

A page fault occurs when a user process requests a page that is not currently present in physical memory. When a page fault happens, the kernel loads the required data from disk into memory. Depending on the hints provided via `posix_fadvise()` or `madvise()`, the kernel decides how much additional data (e.g., readahead) to load alongside the requested page.

This information—whether a specific page from a file is present in memory or not—is managed by the **page table**.

### Page Table

The page table maintains mappings between a page's virtual address and its physical address, along with other necessary metadata. The kernel consults the page table to determine if a page is present in memory. If not, it triggers a page fault, loads the page into physical memory, and creates the appropriate mapping between the virtual and physical addresses.

To understand this better, let's look at what a typical entry in a page table looks like (for x86_64 architecture).

| Bit Position | Name                  | Description                                                                 |
|--------------|-----------------------|-----------------------------------------------------------------------------|
| 0           | Present (P)           | 1 if the page is present in physical memory; 0 triggers a page fault.       |
| 1           | Read/Write (R/W)      | 1 allows writes; 0 is read-only.                                            |
| 2           | User/Supervisor (U/S) | 1 allows user-mode access; 0 restricts to kernel-mode.                      |
| 3           | Page Write-Through (PWT) | Controls write-through caching behavior.                                 |
| 4           | Page Cache Disable (PCD) | Disables caching for this page if set.                                    |
| 5           | Accessed (A)          | Set by CPU when the page is read or written.                                 |
| 6           | Dirty (D)             | Set by CPU when the page is written to (indicates modification).            |
| 7           | PAT (Page Attribute Table) | Used for larger pages to control memory type.                             |
| 8           | Global (G)            | Page is global (not flushed from TLB on context switch).                    |
| 9-11        | Available             | Ignored by hardware; used by the OS (e.g., for swap info).                  |
| 12-51       | Physical Page Address | Bits of the physical frame address (aligned to 4 KiB).                      |
| 52-62       | Available/Protection Keys | Additional OS-use bits or protection keys (if enabled).                   |
| 63          | Execute Disable (XD)  | If set, prevents execution of code on this page.                            |

This table summarizes the key bits in a standard 64-bit PTE on modern Linux systems running on x86_64. The exact layout can vary slightly with kernel configuration, but these are the core fields relevant to page presence, permissions, and fault handling.

## Trying out Random Reads

Let's now experiment with **random reads** to observe how the page cache behaves when accesses are non-sequential. We'll start with the `read()` system call (via Python's file object).

Here's the script: 

```python
import os

with open("/root/pagecache-test/samplefile", "br") as f:
    fd = f.fileno()
    os.posix_fadvise(fd, 0, os.fstat(fd).st_size, os.POSIX_FADV_RANDOM)
    print(f.read(2))
    f.seek(40960)
    print(f.read(2))
```

This code:

Applies the POSIX_FADV_RANDOM hint to minimize readahead.
Reads 2 bytes from the beginning of the file (triggering page 0).
Seeks to byte 40960 (exactly the start of the 10th page, assuming 4 KiB pages).
Reads 2 more bytes (triggering page 10).

After evicting the page cache beforehand and running the script, we check the cache population:

```bash
[root@ip-172-31-30-60 pagecache-test]# vmtouch samplefile
           Files: 1
     Directories: 0
  Resident Pages: 2/5120  8K/20M  0.0391%
         Elapsed: 9.2e-05 seconds
```

While `vmtouch` shows the total count, it doesn't reveal which pages are cached (the -v option can list them but is less readable for quick checks). For more detail, we use `fincore`:

```bash
[root@ip-172-31-30-60 pagecache-test]# fincore samplefile
filename size   total pages     cached pages    cached size     cached percentage
0 10
samplefile 20971520 5120 2 8192 0.039062
```

fincore outputs the indices of cached pages on the line below the headers 0 and 10, This precisely matches our accesses which means kernal loads what data we are accessing into the memory ondemand.

Let's repeat the random access experiment using the `mmap()` system call. Since `mmap()` maps the entire file into the process's virtual address space as a byte array, we can directly access any byte by its index.

I've also added a long `sleep(600)` (10 minutes) at the end; the reason for this will become clear in upcoming sections when we inspect live process mappings and page tables.

```python
import mmap
import time

with open("/root/pagecache-test/samplefile", "r") as f:
    with mmap.mmap(f.fileno(), 0, prot=mmap.PROT_READ) as mm:
      mm.madvise(mmap.MADV_RANDOM)
      print(mm[:2])
      print(mm[40961])
      print(mm[81921])
      time.sleep(600)
```

After evicting the page cache beforehand and running the script (checking while it's sleeping), we observe:

```bash
[root@ip-172-31-30-60 pagecache-test]# vmtouch samplefile
           Files: 1
     Directories: 0
  Resident Pages: 3/5120  12K/20M  0.0586%
         Elapsed: 9e-05 seconds
[root@ip-172-31-30-60 pagecache-test]# fincore samplefile
filename size   total pages     cached pages    cached size     cached percentage
0 10 20
samplefile 20971520 5120 3 12288 0.058594
```

As expected, we see very similar behavior to the read() case. Exactly 3 pages are resident in the page cache. fincore confirms the cached pages are precisely 0, 10, and 20—the pages containing the bytes we accessed.

Now that we've confirmed specific pages are loaded into physical memory on demand, let's go one level deeper: how are these virtual addresses assigned in the process, and how does the kernel map them to physical pages? If you've already guessed—we're talking about page tables. In the next section, we'll visualize and explore the actual page table entries for this mapping.

## Page Tables Vizualization

While exploring ways to visualize page tables and virtual address mappings, I discovered two powerful interfaces exposed by the Linux kernel under `/proc/<pid>/`:

### `/proc/<pid>/maps`

* This file lists all virtual memory areas (VMAs) mapped in a process's address space, providing a high-level overview of how memory is allocated and backed. Here is a sample excerpt (from a running Python process with our mmap example):
  ```txt 
  563f053f5000-563f053f6000 r--p 00002000 ca:01 466252                     /usr/bin/python3.9
  563f053f6000-563f053f7000 rw-p 00003000 ca:01 466252                     /usr/bin/python3.9
  563f19251000-563f1933f000 rw-p 00000000 00:00 0                          [heap]
  7fe72bc00000-7fe72d000000 r--s 00000000 ca:01 10119342                   /root/pagecache-test/samplefile
  7fe72d200000-7fe73a730000 r--p 00000000 ca:01 3086                       /usr/lib/locale/locale-archive
  ```
* Notice the entry for our `samplefile`: a large (20 MiB) read-only shared mapping `(r--s)` starting at virtual address `7fe72bc00000`
* Field Breakdown (in order):

| Position | Field                | Example                          | Description                                                                 |
|----------|----------------------|----------------------------------|-----------------------------------------------------------------------------|
| 1        | Address Range        | `7fe72bc00000-7fe72d000000`      | Start and end virtual addresses of the memory region (in hexadecimal).     |
| 2        | Permissions          | `r--s`                           | Access permissions:<br>• `r` = readable<br>• `w` = writable<br>• `x` = executable<br>• `-` = permission not granted<br>Last character: `p` = private (copy-on-write), `s` = shared |
| 3        | Offset               | `00000000`                       | Byte offset into the backing file from where the mapping starts (hex).     |
| 4        | Device (major:minor) | `ca:01`                          | Major and minor device numbers (hex) of the filesystem containing the file. |
| 5        | Inode                | `10119342`                       | Inode number of the backing file (`0` for anonymous mappings like heap).   |
| 6        | Pathname             | `/root/pagecache-test/samplefile`| Path to the backing file or special identifier (e.g., `[heap]`, `[stack]`, `[vdso]`). |

### `/proc/{pid}/pagemap`

* This is a lower-level, binary interface that provides information about each virtual page in the process's address space.
* One 8-byte (64-bit) entry per page (in order of increasing virtual addresses).
* Allows querying the Page Frame Number (PFN)—the physical page in RAM—along with flags indicating page status.
* Extremely useful for determining if a page is present, swapped, file-backed, or zero.
* Field Breakdown (varies depending on the arch):
| Bits     | Name                              | Description                                                                 |
|----------|-----------------------------------|-----------------------------------------------------------------------------|
| 0–54     | Page Frame Number (PFN)           | Physical frame number in RAM (valid only if bit 63 is set).                 |
| 55       | Swapped                           | Set to 1 if the page is currently swapped out to disk.                      |
| 56       | Present (legacy)                  | Legacy indicator of page presence (often redundant with bit 63).            |
| 57–60    | Reserved                          | Unused or reserved for kernel-internal purposes.                            |
| 61       | File-backed or shared anonymous   | Set to 1 if the page is file-backed or part of shared anonymous memory.     |
| 62       | Soft-dirty                        | Set to 1 if the page has been modified since the last soft-dirty clear.     |
| 63       | Page present                      | Set to 1 if the page is resident in physical memory (PFN is valid).         |
| 64       | Exclusive (KSM)                   | Used by Kernel Samepage Merging for page deduplication.                     |

### Visualizing Page Table Entries for a Memory-Mapped File

To directly observe how the Linux kernel manages page table entries for our memory-mapped file, I wrote a Python script that combines information from `/proc/<pid>/maps` and `/proc/<pid>/pagemap`. This script lets us see exactly which pages are present in physical memory and their corresponding Physical Frame Numbers (PFNs).

```python
import os
import struct

pid = input("Enter the process pid: ")
print("Getting Details of process: " + str(pid))
page_size = os.sysconf("SC_PAGE_SIZE")
print("Ramp Page Size: " + str(page_size))

# Parse /proc/<pid>/maps for your file's VMA start/end
with open(f"/proc/{pid}/maps") as maps:
    for line in maps:
        if "/root/pagecache-test/samplefile" in line:
            print("Map found in the maps file: " + line)
            start, end = [int(x, 16) for x in line.split()[0].split('-')]
            print("Start VA - " + str(start) + ", End VA - " + str(end))
            break

with open(f"/proc/{pid}/pagemap", "rb") as pagemap:
    for va in range(start, end, page_size):
        offset = (va // page_size) * 8
        pagemap.seek(offset)
        entry = struct.unpack("Q", pagemap.read(8))[0]
        present = entry & (1 << 63)
        swapped = entry & (1 << 62)
        pfn = entry & ((1 << 55) - 1)
        print(f"VA {hex(va)}: present={bool(present)},  PFN={hex(pfn) if present else 'N/A'}")
```

**What the Script Does (Step-by-Step)**

1. Input PID – Prompts for the process ID (e.g., from our sleeping mmap script).
2. Get Page Size – Uses `os.sysconf("SC_PAGE_SIZE")` to retrieve the system page size (typically 4096 bytes).
3. Parse `/proc/<pid>/maps` – Scans for the line containing our samplefile, extracts the virtual address range (start–end).
4. Iterate Over Virtual Pages – Loops through every page-aligned virtual address in the mapped region.
5. Compute pagemap Offset – Each page has an 8-byte entry in `/proc/<pid>/pagemap`. `Offset = (VA // page_size) × 8`.
6. Read and Decode Entry – Reads 8 bytes and unpacks as a 64-bit integer. `struct.unpack("Q", pagemap.read(8))[0]`
7. Extract Key Information:
  * Present bit (bit 63): True if the page is currently in physical RAM.
  * Swapped bit (bit 55): Indicates if the page is swapped out (rare for file-backed pages).
  * PFN (bits 0–54): Physical frame number — the actual location in RAM (only valid if present).
8. Print Results – Shows virtual address, presence, and PFN for each page.

Now let's execute our page table visualization script while the `mmap` process is sleeping. The long `sleep(600)` gives us ample time to inspect the live `/proc/<pid>/` directory — once the process exits, the kernel immediately removes it.

First, identify the PID of the running Python process:

```bash
[root@ip-172-31-30-60 pagecache-test]# ps -aux | grep python
root       48385  0.4  1.2 250088  8708 pts/7    S+   09:12   0:00 python3 mmap-with-random-access.py
...
```

We use PID 48385 and run the inspection script:

```bash
[root@ip-172-31-30-60 pagecache-test]# python3 page_table_read.py
Enter the process pid: 48385
Getting Details of process: 48385
Ramp Page Size: 4096
Map found in the maps file: 7f0305600000-7f0306a00000 r--s 00000000 ca:01 10119342                   /root/pagecache-test/samplefile

Start - 139650951806976 End: 139650972778496
VA 0x7f0305600000: present=True, PFN=0xf592
VA 0x7f0305601000: present=False, PFN=N/A
VA 0x7f0305602000: present=False, PFN=N/A
VA 0x7f0305603000: present=False, PFN=N/A
VA 0x7f0305607000: present=False, PFN=N/A
VA 0x7f0305608000: present=False, PFN=N/A
VA 0x7f0305609000: present=False, PFN=N/A
VA 0x7f030560a000: present=True, PFN=0x1a140
VA 0x7f030560b000: present=False, PFN=N/A
....
....
VA 0x7f0305613000: present=False, PFN=N/A
VA 0x7f0305614000: present=True, PFN=0x1b893
VA 0x7f0305615000: present=False, PFN=N/A
.....
```

The kernel creates the full virtual memory mapping upfront (visible in /proc/<pid>/maps), but does not allocate physical memory immediately. Physical pages are allocated on demand. 

## Page Faults.
Now, let's explore what occurs when we access data **not yet present in memory** — this is precisely where **page faults** come into play.

The kernel determines whether an access triggers a page fault by checking the **present bit** (bit 63) in the corresponding page table entry (PTE), which we observed in our earlier `/proc/<pid>/pagemap` inspection. If the bit is 0, a **minor page fault** occurs for file-backed mappings like ours.

On a minor page fault:
- The kernel locates the file-backed page in the **page cache** (or loads it from disk if absent — a major fault in cold-cache scenarios).
- It allocates a physical frame, populates it with the file data, updates the PTE (sets present bit and PFN), and resumes the process.
- For our `mmap()` case with `MADV_RANDOM`, only the faulted page (or minimal readahead) is loaded.

### Measuring the Performance Impact: Cached vs. Cold Access

To directly measure the impact of page faults and page cache misses, I created a simple benchmark that accesses a large contiguous chunk of the file via `mmap()`. This forces the kernel to fault in multiple pages at once.

The script reads approximately **5 MiB** (bytes 0 to 5,242,879) from the mapped file:

```python
import mmap
import time

with open("/root/pagecache-test/samplefile", "r") as f:
    with mmap.mmap(f.fileno(), 0, prot=mmap.PROT_READ) as mm:
      mm.madvise(mmap.MADV_RANDOM)
      a=mm[0:5242879]
```

First run (cold cache – after eviction):
```bash
[root@ip-172-31-30-60 pagecache-test]# time python3 mmap-with-bulk-random-access.py

real    0m1.032s
user    0m0.049s
sys     0m0.000s
```

Second run (warm cache – pages now resident in memory):
```bash
[root@ip-172-31-30-60 pagecache-test]# time python3 mmap-with-bulk-random-access.py

real    0m0.027s
user    0m0.018s
sys     0m0.009s
```

Evict the cache and run again (force cold cache):
```bash
[root@ip-172-31-30-60 pagecache-test]# vmtouch -e samplefile
           Files: 1
     Directories: 0
   Evicted Pages: 5120 (20M)
         Elapsed: 0.000276 seconds
[root@ip-172-31-30-60 pagecache-test]# time python3 mmap-with-bulk-random-access.py

real    0m1.020s
user    0m0.000s
sys     0m0.048s
```

Cold cache runs consistently take around ~1 second this time is dominated by disk I/O and major/minor page faults as the kernel loads ~1280 pages (5 MiB / 4 KiB) from storage into the page cache. Warm cache run completes in ~27 milliseconds over 38 times faster! Here, pages are already resident, so accesses are pure memory operations with minimal overhead.

## Conclusion

Through this series of hands-on experiments, we've demystified the Linux page cache's role in read operations and demand paging. Starting with simple `read()` and `mmap()` calls, we observed how the kernel intelligently populates the cache

In our upcoming session, we'll shift focus to **write operations** and their intricate dance with the page cache. We'll design experiments to explore the same