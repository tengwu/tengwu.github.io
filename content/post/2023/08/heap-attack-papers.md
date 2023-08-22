+++
title = "defense of memory attack"
date = 2023-08-22T10:09:00+08:00
lastmod = 2023-08-22T12:54:23+08:00
tags = ["security"]
categories = ["security", [",", "memory_security"]]
draft = false
toc = true
+++

## 一些广为人之的 {#一些广为人之的}

-   NX
-   ASLR
-   stack canary
-   PIE(Position-Independent Executable)
-   RELRO


## Buffer overflow detection {#buffer-overflow-detection}


### redzone {#redzone}

-   Purify: Fast detection of memory leaks and access errors.[1992]
-   Lightweight bounds checking.[2012]
-   ASan: Addresssanitizer: A fast address sanity checker.[2012]
-   Valgrind: a framework for heavyweight dynamic binary instrumentation.[2007]


### bounds checking {#bounds-checking}

-   Backwardscompatible bounds checking for arrays and pointers in c programs.[1997]
-   CRED: A practical dynamic buffer overflow detector.[2004]
-   Backwardscompatible array bounds checking for c with very low overhead.[2006]
-   Baggy bounds checking: An efficient and backwards-compatible defense against out-of-bounds errors.[2009]
-   Paricheck: an efficient pointer arithmetic checker for c programs.[2010]
-   Heap bounds protection with low fat pointers.[2016]
-   Low-fat pointers: compact encoding and efficient gate-level implementation of fat pointers for spatial safety and capabilitybased security.[2013]
-   EffectiveSan: Type and memory error detection using dynamically typed c/c++.[2018]
-   Softbound: Highly compatible and complete spatial memory safety for c.[2009]
-   Sgxbounds: Memory safety for shielded execution.[2017]
-   Fast and generic metadata management with mid-fat pointers.[2017]
-   MPX: Intel mpx explained: A cross-layer analysis of the intel mpx system stack.[2018]
-   Cup: Comprehensive user-space protection for c/c++.[2018]
-   Framer: A tagged-pointer capability system with memory safety applications.[2019]
-   Pico: A presburger in-bounds check optimization for compilerbased memory safety instrumentations.[2021]


### page protection {#page-protection}

-   efence
-   TailCheck: A Lightweight Heap Overflow Detection Mechanism with Page Protection and Tagged Pointers.[2023]
-   Detect Unintended Memory Access (D.U.M.A.).[2022]
-   DYBOC: A dynamic mechanism for recovering from buffer overflow attacks.[2005]
-   libgmalloc: <https://www.unix.com/man-page/osx/3/libgmalloc/>.[]
-   Windows PageHeap: <https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/gflags-and-pageheap>.[2022]
-   Prober: Practically defending overflows with page protection.[2021]
-   Delta pointers: Buffer overflow checks without the checks.[2018]


## Pointer tagging {#pointer-tagging}

-   SafeC: Efficient detection of all pointer and array access errors.[1994]
-   Cyclone: a safe dialect of c.[2002]
-   Ccured: Type-safe retrofitting of legacy code.[2002]
-   The cheri capability model: Revisiting risc in an age of risk.[2014]
-   Cherivoke: Characterising pointer revocation using cheri capabilities for temporal memory safety.[2019]
-   Baggy bounds checking: An efficient and backwards-compatible defense against out-of-bounds errors.[2009]
-   Delta pointers: Buffer overflow checks without the checks.[2018]
-   HeapCheck: Lowcost hardware support for memory safety.[2022]
-   Pacmem: Enforcing spatial and temporal memory safety via arm pointer authentication.[2022]
-   No-fat: Architectural support for low overhead memory safety checks.[2021]


## Use-After-Free detection {#use-after-free-detection}

-   Cets: Compiler enforced temporal safety for c.[2010]
-   Vik: Practical mitigation of temporal memory safety violations through object id inspection.[2022]
-   xtag: Mitigating use-after-free vulnerabilities via software-based pointer tagging on intel x86-64.[2022]
-   MTE
-   SSM(Silicon Secured Memory)
-   Memory tagging and how it improves c/c++ memory safety.[2018]
-   Undangle: Early detection of dangling pointers in use-after-free and double-free vulnerabilities.[2012]
-   Freesentry: protecting against use-afterfree vulnerabilities due to dangling pointers.[2015]
-   Preventing use-after-free with dangling pointers nullification.[2015]
-   Dangsan: Scalable use-after-free detection.[2017]
-   Bogo: Buy spatial memory safety, get temporal memory safety (almost) free.[2019]
-   Efficiently detecting all dangling pointer uses in production servers.[2006]
-   Oscar: A practical page-permissions-based scheme for thwarting dangling pointers.[2017]
-   Dangzero: Efficient use-after-free detection via direct page table access.[2022]


## Unintialized memory read {#unintialized-memory-read}

-   Purify
-   Valgrind
-   Unisan: Proactive kernel memory initialization to eliminate data leakages.[2016]
-   Safelnit: Comprehensive and practical mitigation of uninitialized read vulnerabilities.[2017]
