---
title: Inside Linux File Descriptors
date: 2025-09-14 12:00 +0530
categories: [OS, Files and File Systems]
tags: [linux, files, operating-systems]
toc: true
description:: Working parts behind file descriptors in Linux systems
---

## Introduction

🔧 If you've worked around Linux systems - whether in a containerized environment or on bare metal - chances are you've come across the term file descriptor (FD). You may have even faced issues where FD limits were exceeded, causing applications to crash. 💥
Most of us only brush past this topic with a basic understanding - "just increase the limit and move on." But there's a lot more happening under the hood.
🚀 This blog dives into the working parts behind file descriptors, aiming to shift the way you look at them and help you understand why they matter so much in day-to-day operations.

## What is File descriptor

A simple definition for file descriptor would be,
A file descriptor (FD) is a unique integer identifier used by the operating system to represent an open file, socket, or other I/O resource in a process.
Let's break this down with a simple C example. In this program, we open a file, write some content to it, and then close it. Before closing, we print the process ID (PID) and add a sleep() call. This pause allows us to inspect the process under /proc/<pid> and observe its file descriptors in action.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>

int main() {
    int fd;
    const char *filename = "simple.txt";
    const char *message = "Hello\n";

    // Open the file using system call open()
    fd = open(filename, O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) {
        perror("open");
        exit(EXIT_FAILURE);
    }

    // Print the file descriptor
    printf("File Descriptor: %d\n", fd);

    // Write "Hello" to the file using the file descriptor using the system call write
    if (write(fd, message, strlen(message)) == -1) {
        perror("write");
        close(fd);
        exit(EXIT_FAILURE);
    }

    // Sleep for 180 seconds to inspect the process
    printf("Sleeping for 180 seconds... (PID: %d)\n", getpid());
    sleep(180);

    // Close the file descriptor using system call close()
    if (close(fd) == -1) {
        perror("close");
        exit(EXIT_FAILURE);
    }

    printf("Message written and file closed successfully.\n");
    return 0;
}
```

I have compiled and executed the code. The process paused at the sleep statement, Now lets get into the proc dir and examine the fd related details, to give an context for all process created by linux which opens an file there will be an int returned which indicated that specific file across the process execution. In our example I have opened an file named simple.txt I am getting into the `/proc/<PID>/fd` directory and listing out the content gives me the following

```bash
/proc/197847/fd$ ls -ltrh
total 0
l-wx------ 1 gokul gokul 64 Sep 14 19:13 3 -> '/..../simple.txt'
lrwx------ 1 gokul gokul 64 Sep 14 19:13 2 -> /dev/pts/3
lrwx------ 1 gokul gokul 64 Sep 14 19:13 1 -> /dev/pts/3
lrwx------ 1 gokul gokul 64 Sep 14 19:13 0 -> /dev/pts/3
```

From the program output, it becomes clear that the file descriptor (FD) printed during execution corresponds to the link you see under `/proc/<PID>/fd/`. This shows that Linux maps open files for a process by creating symbolic links named after their FDs, pointing to the actual files. Once the file is closed, the corresponding FD entry is released.

Additionally, the kernel also tracks the current file offset (write/read position). This information can be inspected in `/proc/<PID>/fdinfo/<FD>`, giving deeper insight into how the kernel manages open files.

```bash
/proc/197847/fdinfo$ cat /proc/197847/fdinfo/3
pos:    6
flags:  0100001
mnt_id: 129
ino:    46443371157320495
```

When we checked the `/proc/<PID>/fdinfo/<FD>`, the last write position was shown as 6. This means that any subsequent write will continue from the next position onward. The system call lseek() can be used to modify this offset, allowing more flexible control over read/write operations.

## FD Table

The kernel maintains this information in per-process file descriptor tables. Each entry in this table points to an entry in the system-wide open file table, which in turn points to the file's inode entry.
The per-process FD table: maps integer FDs (like 3, 4, etc.) to system-wide file table entries.
The system-wide open file table: tracks the open file state (file offset, flags, access mode).
The inode table: stores file system–specific information (permissions, ownership, file type, etc.).

This layered design allows multiple processes to have their own FD pointing to the same underlying system-wide entry, enabling features like shared file access.

## Default FDs

You may have also noticed that every Linux process has three default file descriptors opened by the kernel:

* 0 → stdin (standard input)
* 1 → stdout (standard output)
* 2 → stderr (standard error)

A commonly used shell redirection, for example:

```bash
./my_script 2>&1
```

means: redirect FD 2 (stderr) to FD 1 (stdout). Internally, the shell duplicates the file descriptor, so all writes to both stderr and stdout are directed to the same output (usually the controlling terminal).

## Pratical Example

The C code shown earlier is just a sample - but no matter what language you're using for file operations, Linux always creates a file descriptor (FD) to manage them. In Linux, everything is a file - which means even for a proxy like Nginx, every socket it opens corresponds to an entry in the FD table, resulting in a new FD being created.

Now, imagine running a proxy that handles millions of connections with keep-alive enabled. Each of those connections translates into an open FD that remains active for as long as the connection lives. This is a classic scenario where you can quickly hit the FD limit, leading to bottlenecks or failures if the system isn't tuned to handle such scale.

If you've ever run into the dreaded "Too many open files" error in Linux, you've bumped into file descriptor (FD) limits. File descriptors are a fundamental part of how Linux manages resources, and knowing their limits (and how to tune them) is essential for system engineers, DevOps folks, and developers running large-scale applications.

## FD Limits

1. Per-Process File Descriptor Limit

Each process running on Linux can only open a certain number of file descriptors.

* Soft limit → what a process is allowed by default.
* Hard limit → the ceiling for the soft limit.

You can view your current soft limit with:

```bash
ulimit -n
```

Typically, the default is 1024 per process.

The hard ceiling is controlled by:

```bash
cat /proc/sys/fs/nr_open
```

By default, this is set to 1,048,576 (≈ 1 million FDs), which means a single process can be tuned up to this limit.

2. System-Wide File Descriptor Limit

The kernel also enforces a global cap across all processes combined. 

This is defined in:

```bash
cat /proc/sys/fs/file-max
```

This value is usually in the millions on 64-bit systems and represents the maximum number of file descriptors that the kernel will allocate in total

If your application requires more FDs than the defaults,

Increase the per-process limit via:

```bash
ulimit -n (temporary, per-shell)
/etc/security/limits.conf or systemd unit overrides (persistent)
```

Adjust system-wide limits via:

```bash
/proc/sys/fs/file-max
/etc/sysctl.conf
```

Always test carefully - setting these too high without planning memory usage can destabilize your system.