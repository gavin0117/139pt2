First look at the prompt.
Before you start coding, read the full assignment prompt carefully. This includes all sections, examples, and notes. Every detail matters. The description explains how this project builds on the previous one, introduces new concurrency mechanisms, and outlines the expectations for each part. You are responsible for understanding the entire prompt before you begin.

You will download the provided scaffold file, sharedhash.c Download sharedhash.c. This is the file you will edit and submit for Part 1. Do not rename it, and do not submit multiple files.T he provided scaffold contains all necessary infrastructure, including the full working baseline and shared allocator implementation. Read through the code thoroughly before making any changes.

For Part 2, you will create a copy of your working Part 1 solution and rename it esharedhash.c. You will modify and experiment with this version only. Both files must be submitted at the end of the project.

Be sure to read all instructions and comments carefully. Much of what you need to know is documented directly in the scaffold. Understanding the provided implementation before attempting modifications will save significant time later.

Assignment Revision History
This assignment may be updated during the term. If a newer version is released, it will be clearly labeled (e.g., V002, V003, etc.). Always confirm that you are working from the most recent version before submitting your work. Updates may include clarifications, corrected constants, or improved testing instructions.

Modifying the scaffold. 

Keep the constraints tight. The scaffold, Huffman code, printing code, macro definitions, and the mmap pool size are all fixed. Only umalloc and ufree may be altered, plus the portions of first-fit logic they depend on. The output hash must remain identical, so structural behavior visible to the test harness cannot change. That includes allocation order, block layout, and any side effects that feed into the hash. Treat any difference in the hash as a regression.

Test allocation failures against the unmodified scaffold first; if the baseline succeeds, then your modified version must also succeed. You cannot compensate by expanding the pool or tweaking global definitions. Keep modifications minimal and defensible, and explain in the writeup why each change is necessary for correctness rather than convenience. The justification is graded more heavily than the code itself.

Images in Markdown Files

Some students were having difficulty in getting images to reliably show up in markdown files. First, I didn't expect you to include images, however, if you decide to do so, and you are concerned about them showing up, then you may now just upload the images as a part of the submission.

Current Revision:

V002: Update to allowed changes and images. 
V001: Initial release – expect minor revisions
Assignment Outline
Review the Full Prompt
Read every section in the project description carefully. The project has two major parts:

Part 1: Convert the provided shared-memory multi-process implementation into a threaded version.

Part 2: Experiment with allocator optimizations and concurrency improvements.
Each part builds directly on the last, so do not move ahead until your previous step is fully working.

Download and Inspect the Scaffold

Download sharedhash.c Download sharedhash.cfrom Canvas.

Read through the entire file before editing.

Do not modify any code outside the STUDENT SECTION unless explicitly instructed.

Familiarize yourself with the shared allocator, semaphore usage, and process handling.

Part 1: Convert to a Threaded Implementation
Replace all process-related logic with threads using the pthread API. Each worker thread should process one data block concurrently. You will:

Replace fork() and waitpid() with pthread_create() and pthread_join().

Replace all process synchronization using semaphores with mutexes.

Remove pipes and instead store each thread’s result in a shared array.

Ensure all hash results remain consistent with the original multi-process version.

Confirm that your program compiles and runs correctly with multiple threads.

Part 2: Explore and Optimize Concurrency
Copy your working sharedhash.c to esharedhash.c. Modify this version only. In this part, you will:

Experiment with ways to reduce lock contention in the allocator.

Implement one or more optimizations such as per-thread memory pools, fine-grained locking, or opportunistic allocation strategies.

Measure and compare performance between your baseline (sharedhash) and optimized (esharedhash) versions.

Submit both programs along with a short report describing your changes and observations.

Test Frequently
Compile and test after each significant change using small and large input files. Use timing comparisons and correctness checks to verify progress. Example commands:

gcc -pthread -o sharedhash sharedhash.c
gcc -pthread -o esharedhash esharedhash.c ./sharedhash test_1MB.bin -t ./esharedhash test_1MB.bin -t
Use tools like Valgrind to check for memory errors and leaks during development.

Validate and Prepare for Submission

Remove any debugging prints before submission.

Ensure your programs compile cleanly under Linux with gcc -Wall -Wextra -pthread.

Verify that both versions produce identical final hashes for the same inputs.

Include your performance report (plain text or markdown).

Submit both sharedhash.c and esharedhash.c.

Submit your markdown files with the required analysis


Introduction
In this project, you will continue from your previous managed memory and multiprocessing assignment. That earlier project exposed the challenges of maintaining allocator consistency when multiple processes operate on separate memory spaces. In this follow-up, you will receive a complete working solution as your starting point. The provided version uses a shared heap and synchronization to coordinate concurrent access across processes. Your task is to study this correct but inefficient implementation, then extend and refine it: first by converting it to a threaded version using pthreads, and later by reducing lock contention to improve concurrency and performance.

Memory Management and fork()
In the previous project, your allocator functioned correctly for small tests but failed under heavier multiprocessing workloads. The root cause lies in how fork() manages memory. When a process calls fork(), the operating system creates a new process whose address space is a copy of the parent’s, implemented through copy-on-write. Initially, parent and child share the same physical pages, but once either modifies memory, those pages are duplicated privately. This means that while each process starts with an identical view of the heap, any updates to allocator metadata—such as the free list or block headers—are isolated to that process. Each child believes it is managing the same heap, but in reality, they are modifying separate copies, leading to allocator corruption and crashes.

A potential fix would be to reinitialize the allocator after each fork(), so every process manages its own private heap. However, that does not reflect the model of concurrency we intend to study. The goal here is to implement a true shared-memory system in which multiple processes operate cooperatively on a single heap. This is achieved by placing the allocator’s data structures inside a shared mmap() region, allowing all processes to see and modify the same memory. Once memory is truly shared, synchronization becomes essential: without coordination, simultaneous access would destroy allocator state. The simplest solution is coarse-grained locking, where a single global lock protects all allocator operations. Later, you will extend this design toward finer-grained synchronization and compare the shared-memory process model with the threaded model, where threads naturally share memory within one address space.

 

Project Setup and Getting Started
This project begins with a complete, working solution that demonstrates how to use a shared heap allocator across multiple processes. The provided code already implements shared memory using mmap() and synchronizes allocator operations with a process-shared semaphore. Your role is to study this implementation, understand its structure, and then extend it to explore concurrency and performance under different models. While the program runs correctly, it is intentionally inefficient—this inefficiency is the foundation for the work you will perform in later parts of the project.

1. Downloading and Inspecting the Starting Code
Download the provided scaffold file sharedhash.c from Canvas. This is a fully functional version of the shared-memory Huffman hashing program. You must work directly within this file. Do not rename it, split it into multiple files, or alter its structure beyond the clearly marked student sections. The autograder depends on the file name and format, and changing either may result in a failing grade.

Once downloaded, read through the file carefully before compiling. The file is heavily commented to guide you through the allocator, synchronization primitives, and execution flow. You are expected to understand the control flow from main() through run_multi() and process_block() before modifying anything.


gcc -Wall -Wextra -o sharedhash sharedhash.c
./sharedhash sample.txt
./sharedhash sample.txt -m
The -m flag activates multi-process mode, launching a new process for each 1KB block in the input file. If you compile and run with -DDEBUG=2, the program prints process IDs, letting you confirm that multiple processes are running concurrently.


gcc -DDEBUG=2 -Wall -Wextra -o sharedhash sharedhash.c
./sharedhash sample.txt -m
Expected output should include lines like:


[PID 10435] Block 0 hash: 338324
[PID 10436] Block 1 hash: 881232
[PID 10437] Block 2 hash: 765411
Final signature: 1984967
Each child process computes the hash for one block independently, then writes its result back to the parent through a pipe. The parent accumulates these values and prints the final signature. The important property is that this signature must be identical in both single-process and multi-process modes.

2. Understanding the Shared Memory Allocator
The allocator is built on mmap() to create a shared memory region visible to all processes. Unlike malloc(), which allocates memory from a process’s private heap, mmap() with the MAP_SHARED and MAP_ANONYMOUS flags allows multiple processes to access the same physical memory.

Internally, the allocator maintains a single linked list of free blocks—an embedded free list. Each free block stores its size and a pointer to the next free block, all within the managed memory region itself. This design avoids using any external dynamic allocations. The same region also holds the data allocated for the Huffman tree, heap nodes, and other structures.

This is a first-fit allocator: it scans the free list from the beginning and takes the first block large enough to satisfy the request. If the block is larger than required, it splits it into an allocated region and a smaller free block. When a block is freed, the allocator inserts it back into the free list in address order and attempts to coalesce adjacent blocks.

Because the heap is shared across processes, every operation on the free list is protected by a semaphore. The semaphore itself is stored in shared memory, so that all processes synchronize on the same lock. This ensures correctness but introduces significant performance overhead—only one process at a time can allocate or free memory.


// Simplified pattern (already implemented)
void *umalloc(size_t size) {
    sem_wait(mLock);     // Acquire global lock
    void *ptr = _umalloc(size);  // Perform allocation
    sem_post(mLock);     // Release lock
    return ptr;
}

void ufree(void *ptr) {
    sem_wait(mLock);
    _ufree(ptr);
    sem_post(mLock);
}
Your later tasks will focus on reducing this lock contention, but for now, you should confirm that this mechanism prevents memory corruption and maintains correctness.

3. Building Confidence with Incremental Testing
This project is complex. Treat it as a systems lab, not a single coding exercise. You will be working with mmap(), process control, and synchronization, all of which can fail in ways that are hard to diagnose. Build confidence gradually. Follow this general workflow:

Verify that the provided code compiles without warnings and runs correctly in both single and multi modes.
Use valgrind to check for memory errors and leaks in single mode.
Re-run with -m and verify that the final signature matches the single-process version.
Use ps or top to confirm multiple processes are active concurrently.
Experiment with different input sizes (e.g., 1KB, 100KB, 1MB) to observe performance changes.
For instance:


# Generate test files
dd if=/dev/urandom of=test_1KB.bin bs=1024 count=1
dd if=/dev/urandom of=test_100KB.bin bs=1024 count=100
dd if=/dev/urandom of=test_1MB.bin bs=1024 count=1024

# Test correctness
./sharedhash test_1KB.bin
./sharedhash test_1KB.bin -m

# Test with debugging
gcc -DDEBUG=2 -o sharedhash sharedhash.c
./sharedhash test_100KB.bin -m
If both runs produce the same final signature, your allocator and synchronization mechanisms are functioning correctly. If they differ, the most likely cause is a race condition or corruption in the free list.

4. Debugging Memory Errors
Memory bugs are common in this type of project and can be subtle. Use Valgrind to detect invalid reads, writes, and memory leaks. Remember that Valgrind will not always interpret custom allocators correctly—it can report false positives when you use mmap()-based memory management—but it remains invaluable for finding invalid pointer usage.


valgrind --leak-check=full --show-leak-kinds=all ./sharedhash test_1KB.bin
If Valgrind reports “Invalid read of size 8” or “Invalid write of size 8,” the most common causes are:

Overwriting allocator metadata (e.g., writing into a freed block)
Using unaligned pointers when splitting blocks
Corrupted next pointers in the free list
Failing to acquire the semaphore before calling umalloc() or ufree()
To help you locate these errors, use print statements carefully. For instance, you can temporarily instrument _umalloc() and _ufree() with diagnostic output:


fprintf(stderr, "Allocating %zu bytes at %p (free list head: %p)\n", size, ptr, *free_list_ptr);
Always remove debugging output before submission. The autograder expects output to match the reference format exactly.

5. Asking Good Questions
Asking for help effectively is part of this project’s learning goal. When you get stuck, focus on presenting evidence rather than symptoms. Instead of saying “my code crashes,” describe what you tested, what you expected, and what actually happened. For example:

“I ran ./sharedhash test_100KB.bin -m and confirmed that both the single and multi versions produce different signatures. I added a print statement inside _ufree() and noticed that after freeing the third block, the next pointer points to 0xDEADBEEF. This suggests that my coalescing logic is touching allocated memory.”

That kind of description makes it much easier for an instructor or TA to guide you toward the root cause.

6. Experimentation and Performance Observation
The current implementation uses a single global semaphore to protect the allocator. This guarantees correctness but eliminates most of the performance benefit of running in parallel. You can measure this effect directly using time:


time ./sharedhash test_1MB.bin
time ./sharedhash test_1MB.bin -m
You may notice that the multi-process version is not significantly faster—and may even be slower—than the single-process version. That’s expected. The cost of locking, context switching, and process creation dominates the small parallel gains. This inefficiency is intentional: in the next phase, you will explore strategies to improve concurrency by reducing contention and comparing process-based and thread-based models.

For deeper analysis, you can instrument the allocator to count how often the semaphore is acquired. For example:


#include <stdatomic.h>

atomic_int lock_acquisitions = 0;

void *umalloc(size_t size) {
    atomic_fetch_add(&lock_acquisitions, 1);
    sem_wait(mLock);
    void *p = _umalloc(size);
    sem_post(mLock);
    return p;
}
At the end of main(), print this value to see how many times the allocator lock was acquired. For larger files, you may see tens of thousands of lock acquisitions, showing that nearly every memory operation is serialized.

7. Compilation and Environment Notes
All grading and testing will be performed on Linux using gcc. You may develop on macOS, but note that:

valgrind is unreliable on macOS and may fail entirely.
macOS uses a different implementation of mmap() that sometimes behaves differently with shared memory.
Semaphore semantics are not identical across platforms; sem_init() with pshared=1 is not supported on all macOS versions.
For these reasons, testing your final implementation in the Linux lab (or on a virtual machine running Linux) is mandatory. Platform-specific differences will not be accepted as valid reasons for failing tests.

8. Project Expectations
This project is designed to develop your intuition about shared memory, synchronization, and allocator design. While the codebase looks intimidating, much of it is provided scaffolding. Your focus should be on understanding what each component does and how data flows through the system. Specifically:

Be able to explain how the allocator manages free and allocated blocks.
Understand how the semaphore prevents corruption when multiple processes call umalloc() or ufree().
Recognize why buffer allocation with umalloc() is necessary before fork().
Appreciate that correctness comes before performance—your modifications must always produce the same final signature as the baseline version.




Before you begin making changes, ensure that your copy of the baseline program runs correctly and passes all correctness checks. Only once you have verified that behavior should you move on to experimenting with modifications. As the complexity of your implementation increases, rigorous testing will become more important. Save known-good versions frequently and use git or versioned backups to avoid losing working code.

Finally, understand that this assignment reflects real systems work. Allocators, synchronization primitives, and concurrent data structures are cornerstones of operating systems and runtime libraries. Debugging them is challenging and time-consuming—but this difficulty is what makes them valuable learning experiences.
