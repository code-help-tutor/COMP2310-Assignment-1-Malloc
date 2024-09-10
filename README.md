
这是一个 C 语言内存分配器的作业，主要内容包括：
1. 背景和需求：
    - 管理内存是 C 编程的重要部分，需实现一个受 Doug Lea 的 DLMalloc 分配器启发的内存分配器，允许用户根据需要进行内存分配和释放。
    - 解决动态内存分配的需求，减少内存碎片，提高分配效率。
2. 具体内容：
    - 显式空闲链表：通过在每个块的元数据中添加指针，将隐式空闲链表改进为显式空闲链表，降低分配时间复杂度。
    - 处理内存碎片：通过合并相邻空闲块来解决内存碎片问题。
    - 放置策略：实现最佳适应策略，选择最小的合适空闲块进行分配。
    - 处理块边缘：在内存块的两端插入“防护柱”，以解决块边缘的问题。
    - 优化措施：包括减少元数据占用、实现常数时间合并、使用多个空闲链表和从操作系统获取额外内存块。
    - 性能测量：通过执行时间和碎片率两个指标来评估分配器的性能。
    - 垃圾回收器：实现一个保守的标记-清除垃圾回收器，自动释放不再使用的内存块。
3. 任务要求：
    - 实现内存分配器的各种功能，包括分配、释放和管理额外内存块。
    - 进行性能分析，包括测量执行时间和碎片率，并比较不同优化级别下的性能。
    - 提交报告，描述内存分配器的实现，包括概述、优化措施、测试、基准测试结果等部分。
4. 测试和提交：
    - 使用提供的测试脚本和内部测试进行测试，可添加额外测试。
    - 通过 Gitlab 提交代码和报告，注意提交清单和良好的 Git 操作习惯。
    - CI 管道可用于测试代码，需关注测试结果。


Introduction
Managing memory is a major part of programming in C. You have used malloc() and free() in the recent labs. You have also built a very basic memory allocator, and it is now time to build a more advanced allocator. In this assignment, you will implement a memory allocator, which allows users to malloc() and free() memory as needed. Your allocator will request large chunks of memory from the OS and efficiently manage all the bookkeeping and memory. The allocator we ask you to implement is inspired by the DLMalloc allocator designed by Doug Lea. The DLMalloc allocator also inspired the PTMalloc allocator, which GLibC currently uses. Indeed, our allocator is a simplified version of DLMalloc, but you will also notice many similarities.

Background
We hope that the last two labs have motivated the need for dynamic memory allocators. Specifically, we have seen that while it is certainly possible to use the low-level mmap and munmap functions to manage areas of virtual memory, programmers need the convenience and efficiency of more fine-grained memory allocators. If we managed the memory from the OS ourselves, we could allow allocating and freeing variables in any order, and also reuse memory for other variables.

The last lab taught you how best to build an implicit free list allocator for managing free blocks. In this assignment, we will first build a more efficient free list data structure called an explicit free list. We will also change our placement policy to best fit, and then perform a number of optimizations.

Explicit free list
The block allocation time with an implicit free list is linear in the total number of heap blocks which is not suitable for a high-performance allocator. We can add a next and previous pointer to each block’s metadata so that we can iterate over the unallocated blocks. The resulting linked list data structure is called an explicit free list. Using a doubly linked list instead of a free list reduces the allocation time from linear in the total number of blocks to linear in the total number of free blocks.

Dealing with memory fragmentation
Fragmentation occurs when otherwise unused memory is not available to satisfy allocate requests. This phenomenon happens because when we split up large blocks into smaller ones to fulfill user requests for memory, we end up with many small blocks. However, some of those blocks may be able to be merged back into a larger block. To address this issue requires us to iterate over the free list and make an effort to find if the block we are trying to free is adjacent to another already free block. If neighboring blocks are free, we can coalesce them into a single larger block.

Placement policy
Memory fragmentation is also affected by placement policy, which is how you choose a block to allocate when given a choice between multiple free blocks which are suitable (large enough). The policy we used in Lab 5 was the first fit policy, returning the first free block we find. There are a number of alternative strategies with the aim of targeting fragmentation, sometimes with a tradeoff of performance:

Best fit, returning the smallest suitable free block (leaves bigger free blocks available).
Next fit, which starts looking for a free block from where the previous allocation was (addresses issues with First Fit such as a tendency to allocate memory at the beginning of the chunk and create more fragments there)
The base spec of this assignment requires you to implement best fit. Note that in the most naive explicit free list implementation, this does come at the performance cost of looking through the entire free list! But one of the optimisations we describe below will heavily mitigate this. You may also consider your own ideas for mitigating this cost.

Dealing with the edges of chunks
One detail we must consider is how to handle the edges of the chunks from the OS. If we simply start the first allocable block at the beginning of the memory chunk, then we may run into problems when trying to free the block later. This is because a block at the edge of the chunk is missing a neighbor. A simple solution to this is to insert a pair of fenceposts at either end of the chunk. The fencepost is a dummy header containing no allocable memory, but which serves as a neighbor to the first and last allocable blocks in the chunk. Now we can look up the neighbors of those blocks and don’t have to worry about accidentally coalescing outside of the memory chunk allocated by the OS, because anytime one of the neighbors is a fencepost we cannot coalesce in that direction.

Optimizations (CR-D level)
We will also perform the following optimizations as part of the assignment to improve the space and time complexity of our memory allocator.

Reducing the Metadata Footprint
Naive solution: In our description of the explicit free list above, we assume the memory allocated to the user begins after all of the block’s metadata. We must maintain the metadata like size and allocation status because we need it in the block’s header when we free the object.

Optimization 1: While we need to maintain the size and allocation status, we only use the free list pointers when the object is free. If the object has been allocated, it is no longer in a free list; thus, the memory used to store the pointers can be used for other purposes. By placing the next and previous pointers at the end of the metadata, we can save an additional 2 * sizeof(pointer) bytes and add that to the memory allocated to the user.

Optimization 2: The allocated flag that tells if a block is allocated or not uses only one bit. Since the sizes are rounded up to the next 8 bytes, the last three bits are not used. Instead of using a boolean to store the allocated flag, we can use one of the unused bits in size. That will save an additional 8 bytes.

Constant Time Coalesce
Naive solution: We mentioned above that we could iterate over the free list to find blocks that are next to each other, but unfortunately, that makes the free operation O(n), where n is the number of blocks in the list.

Optimized solution: The solution we will use is to add another data structure called Boundary Tags, which allows us to calculate the location of the right and left blocks in memory. To calculate the location of the block to the right, all we need to know is the size of the current block. To calculate the location of the block to the left, we must also maintain the size of the block to the left in each block’s metadata. Now we can find the neighboring blocks in O(1) time instead of O(n).

Multiple Free Lists
Naive solution: So far, we have assumed a single free list containing all free blocks. To find a block large enough to satisfy a request, we must iterate over all the blocks to find a block large enough to fulfil the request. For best fit, we also need to look through ALL of the free blocks to find the best such block.

Optimized solution: We can use multiple free lists. We create n free lists1, one for each allocation size (8, 16, …, 8*(n-1), 8*n bytes.) That way, when a user requests memory, we can jump directly to the list representing blocks that are the correct size instead of looking through a general list. If that list is empty, the next non-empty list will contain the block best fitting the allocation request. However, we only have n lists, so if the user requests 8*n bytes of memory or more, we fall back to the naive approach and scan the final list for blocks that can satisfy the request. This optimization cannot guarantee an O(1) allocation time for all allocations. Still, for any allocation under 8*n, the allocation time is O(number of free lists) as opposed to O(length of the free list).

Getting Additional Chunks From the OS
The allocator may be unable to find a fit for the requested block. If the free blocks are already maximally coalesced, then the allocator asks the kernel for additional heap memory by calling mmap. The allocator transforms the additional memory into one large free block, inserts the block in the free list, and then places the requested block in this new free block.

Note that for the purposes of performance analysis and calculating fragmentation (see below), and for determining invalid pointers passed to free due to them not lying in any allocated chunk, you will need to implement a way to iterate through all of the chunks and keep track of where they all are. Hint: consider using a linked list and extra fields in the chunk fenceposts.

Performance Measurement (CR-level)
Given the effort that we are going to in order to optimise the performance of our allocator, we would like to know that it is actually having an effect! There are two primary metrics for the performance of the allocator:

Execution time: The raw best/average/worst time it takes to perform an allocation, and to free.
Fragmentation: Measuring the allocator’s ability to efficiently use memory. Divided into two classes: internal fragmentation, created by space inside blocks which is unusable (e.g. extra padding to meet alignment constraints or metadata), and external fragmentation, caused by space being divided up too much and there being small “fragments” in between allocated blocks which cannot satisfy most requests and sit there wasting space.
It is worth noting that these metrics will change based on these inputs:

Number of allocations made so far/how long the program has been running
Allocation and freeing patterns
As you add optimisations and improve your allocator, you should compare your new iterations’ performance to your old one. This will involve keeping different versions of your allocator allowed in different files (we will advise how to run tests on the allocator given in a specific file in the testing section). You are required to at least analyse the performance benefit arising from your “best” optimisation, e.g. multiple free lists, by comparing a version of the allocator with and without it. You will be asked to analyse this in your report.

Note that if you don’t keep any prior versions, this is also fine, you will just have to make a baseline allocator to compare against (by copying the current allocator to a new file and undoing your best optimisation) yourself.

Measuring execution time and fragmentation
Measuring execution time is straightforward using the Linux time command on a program which is making some number of allocations.

Measuring fragmentation is more tricky. The lectures describe a metric called peak memory utilisation which you can use for this; see Slides 12-14 in the “malloc-basic” half of Week 6’s lectures for details. This will require you to sum up the payload sizes across all blocks, which requires iterating through all of the blocks (in each chunk) in the first place; we will have you define helper functions for doing this. You can also develop your own metrics for measuring fragmentation if you wish.

We will provide you benchmarking programs on which you can measure these metrics and see how they scale with number of allocations. See the benchmarking section for more.

Garbage Collector (HD-level)
A critical pitfall of the malloc/free manual style of memory management is that if the user forgets to free memory, it results in a memory leak, i.e., memory is consumed even if the block is no longer used or not even reachable by the program. The garbage collector (GC) is an alternative to manual freeing that takes this burden off the user by automatically figuring out if allocations are still being used, and freeing those which are not (i.e., dead memory). You will implement what is called a “conservative mark-and-sweep GC” in this assignment.

Basic idea
The basic idea is simple. The program has a bunch of variables which it can access directly by name which we call the roots. The roots consist of local variables as well as global variables (though for simplicity we will ignore the latter for this assignment). These roots can then be pointers to objects on the heap, which may themselves be pointers, and so on. Any object which cannot be reached by a series of pointer dereferences starting from the roots is unreachable and hence garbage, and can safely be deallocated. Meanwhile, if an object IS reachable, then it may still be used later in the program and we cannot free its memory.

Our GC will be a function that frees all blocks that do not contain any reachable objects. The GC consists of two phases: mark and sweep. In the mark phase, we start from the roots and look for pointers to locations on the heap. If we find a pointer that points to a location within a block on the heap, we “mark” that block as reachable by changing a flag in the header (similar to the allocated flag). Once done, we then go through and free all of the allocated blocks that were not marked in the “sweep” phase.

The mark flag can be implemented as one of the unused bits at the end of the size parameter in your header.

Modifying malloc and free
Your malloc and free will need to be modified a bit so that they maintain a data structure tracking the allocated blocks and the locations of their headers. The reasons are twofold:

In the mark phase, if you have a pointer to the middle of a block, you will need to then find the header of the block in order to set the mark flag. This can be done by going through all the allocated block headers and choosing the one corresponding to the block containing the pointer’s location.
In the sweep phase, you need to go through each of the allocated blocks when freeing the unmarked ones.
One way is to keep next/prev pointers in the headers of allocated blocks and having your malloc/free maintain a “used list”, similar to the free lists you’ve used so far. Of course, this uses extra metadata, and you need to go through all of the allocated blocks just to find the header of a single one. You are encouraged to use a better approach if you can think of one.

Conservative GC
Now, the idea of an “object” in C is quite loose, and there are two problems:

Any value can be cast to a pointer and dereferenced. So even if a variable is of type int, it has to be treated as a pointer.
A pointer points to a single location in memory, but using pointer offsets we could also access nearby locations. E.g. if the pointer points to an array we’d access the elements of that array via offsets. Given a pointer, you can’t tell where the object pointed to begins and ends.
This is why we will make our allocator “conservative”, which means that it may fail to free some blocks which are actually garbage, but it will never free a block which may be used. In practice:

When checking the local variables/roots for pointers we scan the entire stack, casting all word-aligned values to pointers (as any of them may be interpreted as pointers).
When a pointer to a block is found, we don’t just check the value being pointed to but all other word-aligned values in that block for further pointers (as those values can be accessed by offsets from the original pointer).
In order to have a more “precise” GC, you need more support from the programming language and the compiler to provide constraints on what objects can be and how you define them, which allows things like having metadata inside the objects themselves so that we can mark individual objects rather than blocks. GCs in practice tend to be built into the programming language itself, such as in Java. You may learn more about this if you take a compiler course in future.

Lab Specification
Malloc spec
You can read the malloc interface on the malloc man page. Many details are left up to the library’s authors. For instance, consider the many optimizations we mention above. All versions of malloc would be correct by the specification on the man page, but some are more efficient than others.

We will also require you to define a set of malloc helper functions. The reasons for these are two-fold:

The helper functions can be used to implement “internal” tests that allow writing unit tests for sub-parts of the malloc. The helper functions also allow probing the “free space” currently available to malloc and assessing fragmentation without knowing the internal implementation details. This will be used in both tests and benchmarking.
Splitting up code into helper functions is good style, and you will find many of these helper functions useful for modularising your code.
Our Implementation Spec
The major elements of the base spec of our malloc are as follows:

Use an explicit free list
Use the best fit placement policy
We will also implement special behavior for my_free to do no-op in cases where the pointer being freed is obviously invalid, i.e. does not correspond to a malloc call. We don’t require handling all such cases, e.g. pointers into the middle of blocks, but the following cases should be handled gracefully:

free is called when the memory allocator has not been initialised
free is called with a pointer which is not in the range of any of the chunks mmapped by the allocator
We have described the basic implementation we want you to follow with optimizations in the background and optimization sections above. We now provide the technical specification of the required design. Some of the requirements are in place to enforce conformance to the design, and others guarantee determinism between our reference allocator and your allocator for testing. The specification below should contain all the details necessary to ensure your implementation is consistent with the reference implementation.

Data Structures and Constants
We provide certain constants, namely:

kMemorySize: When requesting a chunk of memory from the OS for allocation, we always get a multiple of 64MB. For objects larger than 64 MB, you will have to use a multiple of 64 MB.
kAlignment: We require word-aligned addresses from our allocations.
kMinAllocationSize: We initially set the minimum allocation size for our allocator to be 1 word. You are allowed to change this to accommodate the metadata reduction optimisations.
kMaxAllocationSize: We set the maximum allocation size for our allocator to be 128 MB - size of your meta-data. Note that this is the maximum allocation size an individual request to my_malloc. It is possible for the total size across all allocations to be greater than kMaxAllocationSize.
N_LISTS: We set the number of free lists in to be 59. This is only relevant when implementing the “Multiple Free Lists” optimisation. You are allowed and encouraged to change this while debugging/testing performance to see its impact on your allocator.
Marking and Grading Criteria
You are required to submit a completed and working implementation of malloc according to this spec, as well as a report discussing your implementation. The marks will be split in the following way:

Code (60%)
Report, including performance analysis (40%)
The following description of grading categories assumes you submit both the code for your malloc implementation and the report.

Keep in mind that just attempting the given tasks for a grade band is not enough to guarantee you receive a final mark in that grade band. Things like correctness of your implementation, report quality and code style will all influence your results. This is just provided as a guideline.

P
You will be rewarded a maximum grade of P if you complete the following tasks.

Implement a single explicit free list with best fit placement policy
Linear time coalescing
Fence posts
To be able to fulfil kMaxAllocationSize requests after you have implemented fence posts, you can either:

Modify the kMaxAllocationSize constant to subtract the size taken up by the fence posts.
Modify your code to calculate how many multiples of 64MB the call to mmap should request, so that it takes the size of the fenceposts into consideration.
CR
You will be rewarded a maximum grade of CR if you complete the following tasks.

All tasks in the P category
Metadata reduction
Constant time coalescing with boundary tags
Requesting additional chunks from the OS
Performance analysis on the effect of the extensions in this category
D
You will be rewarded a maximum grade of D if you complete the following tasks.

All tasks in the P and CR categories
Multiple free lists
Performance analysis on the effect of the extensions in this category
HD
You will be rewarded a maximum grade of HD if you complete the following tasks.

All tasks in the P, CR and D categories
The garbage collector
Detailed specification
Here, we elaborate further on the exact specification for each task.

Allocation
An allocation of 0 bytes should return the NULL pointer for determinism.
All chunks requested from the OS should be a multiple of size kMemorySize defined in mymalloc.h.
All requests from the user are rounded up to the nearest multiple of kAlignment (8 bytes).
The minimum request size is the size of the full header struct. Even though the pointer fields at the end of the header are not used when the block is allocated, they are necessary when the block is free, and if space is not reserved for them, it could lead to memory corruption when freeing the block.
When allocating from the final free list (N_LISTS - 1), the blocks are allocated in best-fit order: you will iterate the list and look for the smallest block large enough to satisfy the request size. Given that all other lists are multiples of 8, and all blocks in each list are the same size, this is not an issue with the other lists.
When allocating a block, there are a few cases to consider:
If the block is exactly the request size, the block is simply removed from the free list.
If the block is larger than the request size, but the remainder is too small to be allocated on its own, the extra memory is included in the memory allocated to the user and the full block is still allocated just as if it had been exactly the right size.
If the block is larger than the request size and the remainder is large enough to be allocated on its own, the block is split into two smaller blocks. We could allocate either of the blocks to the user, but for determinism, the user is allocated the block which is higher in memory (the rightmost block).
When splitting a block, if the size of the remaining block is no longer appropriate for the current list, the remainder block should be removed and inserted into the appropriate free list.
When no available block can satisfy the user’s request, we must request another chunk of memory from the OS and retry the allocation. On initialization of the library, the allocator obtains a chunk from the OS and inserts it into the free list. The pointer to the new chunk should be saved somewhere in a way that allows traversing across all of the chunks in some order.
In operating systems, you can never expect a call to the OS to work all the time. If allocating a new chunk from the OS fails, your code should return the NULL pointer, and errno should be set appropriately (check the man page).
The allocator should allocate new chunks lazily. Specifically, the allocator requests more memory only when servicing a request that cannot be satisfied by any available free blocks.
Deallocation
Freeing a NULL pointer is a no-op (don’t do anything).
When freeing a block, you need to consider a few cases:
Neither the right nor the left blocks are unallocated. In this case, simply insert the block into the appropriate free list
Only the right block is unallocated. Then coalesce the current and right blocks together. The newly coalesced block should remain where the right block was in the free list
Only the left block is unallocated. Then coalesce the current and left blocks, and the newly coalesced block should remain where the left block was in the free list.
Both the right and left blocks are unallocated, and we must coalesce with both neighbors. In this case, the coalesced block should remain where the left block (lower in memory) was in the free list.
When coalescing a block, if the size of the coalesced block is no longer appropriate for the current list, the newly formed block should be removed and inserted into the appropriate free list. (Note: This applies even to cases above where it is mentioned to leave the block where it was in the free list.)
my_free should behave gracefully (i.e. not crash) when given obviously invalid memory addresses. For the purposes of this requirement an invalid memory address is one which is not in the range of any of the chunks mmapped by the my_malloc.
Garbage collection
Garbage collector uses a modified version of malloc that creates a “used list” or other data structure to track allocated blocks, potentially using extra metadata in blocks to do so. Your free will also need to remove blocks from this data structure.
Garbage collection is a function that can be called by the program anytime, like “free” except you don’t need to tell it what to free (it will figure it out by itself)
GC’s mark phase scans the entire stack and considers any word-aligned value in the stack as a pointer
You are already provided a function get_end_of_stack() that returns a pointer to the end of the stack, and there is also a global variable start_of_stack which you can assume has been pre-initialised to the start of main()’s stack frame. The use of these two is demonstrated in the template file for GC that you’re provided.
GC ignores global variables and variables stored in registers
If a pointer to a block is found, the entire block is scanned for further pointers
GC is resistant to circular references and mark phase does not re-scan a block that it’s already scanned. E.g. if block 1 contains a pointer to block 2, which contains a pointer back to block 1, your mark phase must not fall into an infinite loop.
When sweeping/freeing unused blocks, garbage blocks are removed from the used list (or analogous data structure) and added to the appropriate free lists, and the same rules for coalescing described for free are applied.
Tasks
Your task is to implement malloc (memory allocator) and include in your implementation the various requirements and optimizations discussed above. Broadly, your coding tasks are three-fold.

Allocation
Calculate the required block size.
Find the appropriate free list to look for a block to allocate.
Depending on the size of the block, either allocate the full block or split the block and allocate the right (higher in memory) portion to the user.
When allocating a block, update its allocation status.
Finally, return the user a pointer to the data field of the header.
Deallocation (Freeing)
Free is called on the same pointer that malloc returned, which means we must calculate the location of the header by pointer arithmetic. Hint: You will most likely want to use the ptr_to_block helper function for this.
Once we have the block’s header freed, we must calculate the locations of its right and left neighbors, using pointer arithmetic and the block’s size fields.
Based on the allocation status of the neighboring blocks, we must either insert the block or coalesce it with one or both of the neighboring blocks.
Managing additional chunks
Handle the case where the user’s request cannot be fulfilled by any of the available blocks. Also implement a method for traversing across all of the chunks currently used by the allocator, for use in calculating fragmentation and detecting invalid frees.

Note that the tests we provide will succeed even if you submit an mmap or an implicit free list allocator. The success of these provided tests on a non-explicit free list allocator does not mean you are done. Do not submit code files with allocators from a previous lab. We have tests to ensure compliance with the assignment specification.

Garbage Collection
Modify malloc and free to maintain additional data structures your GC needs
Iterate from the beginning to the end of the stack. For each value in the stack, find if that value is an address to a location inside an allocated block. If such a block is found, set its mark flag.
Recursively scan each block marked in step 2 for pointers to further blocks until no more blocks can be marked.
Go through all allocated blocks and call free for any remaining unmarked blocks.
Performance analysis
Implement at least the peak memory utilisation metric for fragmentation.
Measure and compare execution time and fragmentation for various levels of optimisation of your allocator, and for various input sizes. At least comparing your “best” allocator with a “second best” (a version missing the best optimisation you implemented)
Report
You must submit a report that describes your malloc implementation. It should be written in the given report.md file as valid markdown and split up into the following sections. Some of these will be omitted if you did not complete those specific optimisations:

Overview of your memory allocator, and the data structures used. (100-300 words)
You should make it clear what optimisations you have attempted in this Overview section, but leave the important details until the relevant optimisation section.
You should include a high level explanation of how your best fit allocation strategy operates, how coalescing is implemented, and how fenceposts were implemented.
Optimisations: for each of the following optimisations you attempt, include a section going into more detail about how you implemented it and what effect (if any) it had on your memory allocator:
Metadata Reduction (50-100 words)
Constant time coalescing with boundary tags (50-150 words)
Requesting additional chunks from the OS (50-150 words)
Multiple Free Lists (50-200 words)
Garbage Collector Overview: (100-500 words)
If you implemented the garbage collector provide an overview of how your garbage collector works and the changes you made to the metadata stored.
Describe any problems with implementing a garbage collector in C, and why garbage collectors in general require more support from the programming language itself.
Testing: (100-300 words)
You should include at least one of the following:
Explanation of an implementation challenge encountered in completing this assignment. This could include a difficult bug you faced or a conceptual problem you got stuck on.
Explanation of additional tests you created to verify your malloc and/or garbage collector implementation
If there are any known bugs in your implementation, list them here in this section.
Benchmarking Results: (100-300 words)
Include a summary of your memory allocators performance under the benchmark script. Identify any optimisations you performed that improved the results significantly.
Include any measurements of memory fragmentation you computed.
The report job of the CI will fail if you exceed 1200 words. If your report is still within the word count indicated above (2000 words), you are allowed to safely ignore this failure.

Marking
The code is worth 60% of your grade (in your specific category). The report is worth 40% of the grade.

Note that having appropriate code style will contribute a small amount to your code mark. This means you should:

Have clear names for functions and variables
Have comments explaining what more complex functions do
Remove any commented out code before submission
Remove excessive whitespace
Have consistent whitespace
Make sure you only use the LOG macro to print things to stdout/stderr, so these are automatically removed when a release build is used.
Coding and Implementation
Fork the Assignment 1 repo and then clone it locally.

mymalloc.h
This file contains the type signatures of my_malloc and my_free, various required helper functions and some pre-defined constants. Importantly this will contain the initial definition of the Block data structure. You are allowed to modify this file as needed, but it is your responsibility to make sure the changes are compatible with the CI and the tests.

mymalloc.c
This file will contain your implementation of the my_malloc and my_free functions. We only provide some constants to help with your implementation. Your task will be to implement an explicit free-list allocator. We recommend using a modular approach with judicious use of helper functions as well as explanatory comments. You can insert logging calls with the LOG() macro we provide.

Helper functions
We provide a number of helper functions in mymalloc.h and mymalloc.c:

int is_free(Block *block): Return 1 if the given block is free, 0 if not
size_t block_size(Block *block): Return the size of the given block
Block *get_start_block(void): Returns the first block in memory (excluding fenceposts). If there are multiple chunks, return the first block in the first chunk.
Block *get_next_block(Block *block): Return the next block, contiguously, in memory. If this is the last block of a chunk, return the first block of the next chunk. If this is the last chunk, return NULL.
Block *ptr_to_block(void *ptr): Given a ptr assumed to be returned from a previous call to my_malloc, return a pointer to the start of the metadata block.
Many of these helper functions will help you in writing various versions of the allocator. These also provide an interface for accessing and testing the internals of the allocator, as well as benchmarking. In particular, the get_start_block and get_next_block helpers will help you calculate fragmentation.

Various optimisations will require you to change your implementation of these helpers:

is_free and block_size will change when you implement metadata reduction. This will also make these functions non-trivial (and actually useful).
get_start_block and get_next_block are sensitive to adding the multiple chunks feature. You need to track both a “first” chunk and a way to go from a given chunk to the “next” chunk.
mygc.h
This file contains type signatures for some additional functions that are described immediately below.

mygc.c
This file will contain your implementation of the modified versions of my_malloc and my_free to add extra elements to facilitate the GC, as well as your implementation of my_gc, the garbage collector. You are recommended to first copy in your implementations of my_malloc and my_free from the main C files, as well as any constants and helper functions you had defined there. We also provide the function get_end_of_stack() and the external variable start_of_stack (which we will set at the beginning of our test functions to correspond to the start of stack) to give you the endpoints of the stack for convenience.

mygctest.c
A file with a template main() function where you can insert calls to your GC and test that it frees garbage, avoids freeing reachable blocks, and so on. You are advised not to modify the initialisation of the start_of_stack variable in this file and use the provided my_calloc_gc. Add whatever you want in this file otherwise for testing your GC.

Note that if you define your tests in seperate functions in this file and call those functions in main(), then because the GC includes all functions in the call stack when scanning the stack, it will include any variables you defined in main() when scanning. So be careful of that.

test.py
Script for testing your implementation.

python3 test.py -h
usage: test.py [-h] [-t TEST] [--release] [--log] [-m MALLOC]

options:
  -h, --help            show this help message and exit
  -t TEST, --test TEST  test name to run
  --release             build in release mode
  --log                 build with logging
  -m MALLOC, --malloc MALLOC
                        allocator name, default to "mymalloc"
The most important option is -t <TEST> which allows you to test your implementation with a single test.

tests/
Directory with test source files and built executables. Some tests in this directory also contain explanations of what the test is doing, and potential reasons you may be failing it. You are allowed (and encouraged!) to write your own additional tests and add them to this file.

bench.py
Script for benchmarking your implementation. The script uses a simple benchmark from the glibc library which stresses your implementation.

usage: bench.py [-h] [-m MALLOC] [-i INVOCATIONS]

options:
  -h, --help            show this help message and exit
  -m MALLOC, --malloc MALLOC
                        allocator name, default to "mymalloc"
  -i INVOCATIONS, --invocations INVOCATIONS
                        number of invocations of the benchmark
The default number of invocations for the benchmark is 10. If you want to perform quick benchmark runs, then you can change the number of invocations to 3 using -i 3. It is recommended to use at least 10 invocations if you are reporting results, however.

bench/
Directory with benchmark source files and built executables.

Testing
You can test your implementation of malloc (assuming you have implemented your allocator in the “mymalloc.c” file) by simply running:

python3 ./test.py
The above command will clean previous outputs, compile your implementation, and then run all the provided tests against your implementation. It will run all tests contained in the tests/ folder of your repo. If you want to run a single test (such as align) then you can run the test script like so:

python3 ./test.py -t align
If you have another implementation in a different file named “different_malloc.c” (for example), then you can run tests using this implementation with:

python3 ./test.py -m different_malloc
This may be useful if you want to test a different implementation strategy or want to benchmark two different implementations (Note: bench.py has this same flag).

Note that while you are allowed to include multiple implementations of the memory allocator in different files, only the one in mymalloc.c will be used when running the CI tests.

If you have inserted logging calls using LOG(), then you can compile and run tests with logging enabled like so:

python3 ./test.py --log
Note that some tests which compare output may fail if logging is enabled!

If you want to run a single test directly (for example align) then you can run it like so:

./tests/align
If you have logging enabled and want to save the log for a particular test (for example align) to a file then you can run the following:

./tests/align &> align.log
Make sure you don’t accidentally add the log file to your git repo as these can get quite large in size!

You will almost certainly require using gdb at some point to debug your implementation and failing tests. You can run gdb directly on a test (for example align) like so:

gdb ./tests/align
By default the tests and your library are built with debug symbols enabled so you don’t have to fiddle with enabling debug symbols to aid your gdb debugging.

Note also that you are allowed to (and recommended to) add extra tests for more complex cases if you like. You can do this by adding extra files in the tests directory containing tests, following the same format as the existing tests (make sure to use the mallocing and freeing wrappers provided in testing.h as well). These extra tests should be automatically included when running test.py. Writing extra tests is especially helpful if you find the benchmark script bench.py to be crashing.

Internal testing
In addition to tests which verify the malloc interface itself works, and things like data consistency (data written to a block stays consistent and does not interrupt metadata), we provide a few tests which use the helper functions you’ve written to verify the internal workings of malloc and/or particular parts of it. This should help further pinpoint errors in your code in some simple cases. Note that these tests require you to have used the helper functions we provided you and implemented them correctly. These tests reside in the internal-scripts directory and are also run by test.py, and you can add more yourself too.

Note that it is possible for you to have designed an optimisation in a way which causes one of these tests to fail even when nothing is actually wrong due to the test expecting certain internal behavior. If this occurs, as long as you are confident it is not an implementation issue, it is not a problem; just mention this in your report.

Address sanitizer
The Makefile we have provided uses the GCC flag -fsanitize=address. This enables the AddressSanitizer which instruments mmeory accesses to detect memory errors at runtime, such as access to unallocated memory. You can read more about it here: https://gcc.gnu.org/onlinedocs/gcc/Instrumentation-Options.html. You may see output from “asan” in your test output as a result of this. Hopefully this will help you debug some of your memory-related bugs in this assignment.

Testing the GC
The test.py script does not include tests for the GC, but you can write your own tests in mygctest.c if you’d like. You will have to modify the Makefile to compile and run the GC tests you write yourself.

There will be additional test cases we use when marking your code that are not given to you in the assignment repository for both mygc and mymalloc.

Benchmarking
Benchmarking execution time
If you want to benchmark your code, then you will have to install some python libraries, namely numpy and scipy. This can be achieved by using pip3, python’s package manager:

pip3 install numpy scipy
This will install the two libraries to your local user (as opposed to system-wide). This is the recommended method for installing per-user packages for python.

Benchmarking your code works in a similar way as testing:

./bench.py
This will run the provided benchmark (glibc-malloc-bench-simple) from glibc 10 times and provide you the average. If you are benchmarking, it is highly recommended to close all other intensive applications on your machine as you may get random interference otherwise. If you want to report your benchmark numbers, it is important to note what CPU and memory speed you were using in your report.

Just like the test script, you can switch the malloc implementation using the -m <MALLOC> flag. This is useful as you may want to have two different implementations that you want to compare performance on.

If you have time, you might also find interesting results by experimenting with the sizes/number of mallocs done by the benchmark.

Benchmarking fragmentation
For benchmarking fragmentation, we provide the internal test internal-scripts/fragmentation.c. This tests provides a number of random allocations and frees. There is then a space in the code containing a TODO where you are able to calculate fragmentation and report it. You can run this script the same way as the other tests (test.py).

As described earlier, how exactly you calculate fragmentation is up to you, but we require that you implement at least the peak memory utilisation metric described in lectures.

If you have time, you might also find interesting results by experimenting with the compile time constants at the top of the file.

Submitting your work#
Submit your work through Gitlab by pushing changes to your fork of the assignment repository. A marker account should automatically have been added to your fork of the Assignment 1 repo (if it isn’t there under “Members” then let one of your tutors know).

We recommend maintaining good git hygiene by having descriptive commit messages and committing and pushing your work regularly. We will not accept late submissions.

Submission checklist
The code with your implementation of malloc in mymalloc.c.
The code with your implementation of the GC in mygc.c
The report.md is in the top-level directory.
(Optional) Any optional tests and benchmrks you want us to look at.
Your statement-of-originality.md has been filled out correctly.
Gitlab CI and Artifacts
For this assignment, we provide a CI pipeline that tests your code using the same tests available to you in the assignment repository. It is important to check the results of the CI when you make changes to your work and push them to GitLab. This is especially important in the case where your tests are passing on your local machine, but not on the CI - it is possible your code is making incorrect assumptions about the machine your memory allocator is running on. If you’re failing tests in the CI then it is best to have a look at the CI results for these tests and see if they are giving you hints as to why.


To view the CI pipeline for your repo, click on the little icon to the right of your most recent commit.
# COMP2310 Assignment 1 Malloc

# 程序代做代写 CS编程辅导

# WeChat: cstutorcs

# Email: tutorcs@163.com

# CS Tutor

# Code Help

# Programming Help

# Computer Science Tutor

# QQ: 749389476
