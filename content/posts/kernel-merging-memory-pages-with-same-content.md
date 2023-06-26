---
title: "Kernel Merging Memory Pages With Same Content"
date: 2018-08-26T09:58:03+02:00
draft: false
---

# Merging Memory Pages with Same Content: An Exploration of Kernel Same Page Merging (KSM)

Since Linux Kernel version 2.6.32, there has been a mechanism for the de-duplication and sharing of memory pages, known as Kernel Same Page Merging (KSM). KSM allows for dynamic sharing of identical pages found in different memory areas, even those not shared by the `fork` system call.

When KSM is active, a daemon known as `ksmd` operates in ring-0 (the highest privilege level in x86 CPUs). This daemon scans memory areas marked as mergeable, and when identical content is found, `ksmd` replaces the duplicate pages with a single read-only, write-protected page. This shared page is then mapped into the original locations.

Applications can inform the kernel which pages are mergeable by using the `madvise` system call with the `MADV_MERGEABLE` flag.

## Testing the KSM Feature

A small C program was used to test this kernel feature. This program creates three private pages, fills them with zeroes, and then marks them as mergeable candidates. The `ksmd` daemon has time to scan and merge these pages.

The program then modifies the content of these pages, setting them to a pseudo-random numeric value. As these pages are now read-only due to the merging process, attempting to write to them raises an MMU exception. The kernel handles this exception, and the original page is copied into a new location with write permissions.

Here is the C program used:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <unistd.h>
#include <fcntl.h>

#define N_PAGES 3

void p_s(void) {  
    int fd;
    char buff[1];
    sleep(3);
    fd  = open("/sys/kernel/mm/ksm/pages_sharing", O_RDONLY);
    read(fd, &buff, 1);
    printf("Sharing pages: %d\n", atoi(buff));
    close(fd);
}

void main(void) {  
    int i;
    size_t p_size = sysconf(_SC_PAGE_SIZE);

    void **pages = (void **)calloc(N_PAGES, sizeof(void *));

    // Create 3 pages and fill them with zeroes
    for(i = 0;i < N_PAGES; i++) {
        ((pages[i] = mmap(NULL, p_size, PROT_READ | PROT_WRITE | PROT_EXEC, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0)) && (pages[i])) ?
             memset(pages[i], 0, p_size) && madvise(pages[i], p_size, MADV_MERGEABLE) : exit(-1);
    }

    p_s();
    print("modifying pages contents");

    // Modify the pages to some "random" value
    for (i = 0; i < N_PAGES; i++){memset(pages[i], i+1337, 1);};
    p_s();
}
```

After compiling, the resulting stdout demonstrates the merging and subsequent unmerging process:

```bash
$ ./a.out
Sharing pages: 3  
modifying page contents  
Sharing pages: 0  
```

KSM parameters can be reviewed by running:

```bash
$ sudo grep . /sys/kernel/mm/ksm/pages_*
/sys/kernel/mm/ksm/pages_shared:0
/sys/kernel/mm/ksm/pages_sharing:0
/sys/kernel/mm/ksm/pages_to_scan:100
/sys/kernel/mm/ksm/pages_unshared:0
/sys/kernel/mm/ksm/pages_volatile:0
```

One may wonder if this memory page merging

can be applied to any memory page, even without being explicitly advised by the program itself. The answer is yes, but this requires Ultra KSM (UKSM), a set of patches not yet merged into the mainstream kernel as of writing this.

In conclusion, KSM is a powerful feature provided by the kernel for optimizing memory consumption between processes that share the same data. However, it's worth noting that this is also an expensive process in terms of CPU usage.

In a follow-up article, I'll explore the use of these primitives in Golang.

## References

- [Madvise man page](https://man7.org/linux/man-pages/man2/madvise.2.html)
- [Mmap man page](https://man7.org/linux/man-pages/man2/mmap.2.html)
- [KSM wikipedia](https://en.wikipedia.org/wiki/Kernel_SamePage_Merging)

