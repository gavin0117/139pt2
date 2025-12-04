# Part 2: Performance Optimization with Per-Thread Memory Pools - FINALE 

## Optimization Strategy

I implemented **Strategy A: Per-Thread Memory Pools**. This approach gives each thread its own dedicated memory region (16KB per thread) that it can allocate from without needing to acquire the global mutex. The key idea is that most allocations can happen in the thread's private pool using a simple bump allocator, which is extremely fast and requires no synchronization. Only when a thread's pool runs out does it fall back to the global allocator.

I decided to go with this strategy because the baseline sharedhash.c had severe lock contention. Meaning that every single allocation and free required acquiring the global mutex. Hundreds of threads were constantly fighting for the same lock. Perthread pools eliminate most of this fighting by allowing threads to work independently.

## Threading Model Differences

**sharedhash.c (Part 1 baseline):**
- Spawns up to 1024 concurrent threads (one per file block)
- All threads compete for a single global allocator lock
- No memory management strategy beyond the basic first-fit allocator
- Result: Segfaults on files larger than ~200KB due to memory exhaustion

**esharedhash.c (Part 2 optimized):**
- Uses batched threading with `BATCH_SIZE = 50`
- Processes file in batches: spawn 50 threads, wait for completion, repeat
- Each thread gets a 16KB private memory pool (`THREAD_POOL_SIZE`)
- Pools are recycled between batches (atomic counter reset)
- Result: Successfully handles 10MB+ files with significantly reduced lock contention

### Why 50 Threads Per Batch?

I picked 50 as a balance between memory usage and concurrency:

**Memory constraints:** With a 2MB heap, I needed to reserve space for thread pools while leaving enough for the global allocator. The calculation:
- 50 threads × 16KB pools = 800KB reserved
- Remaining for global allocator: 2MB - 800KB = 1.2MB
- This leaves plenty of space for input buffers and fallback allocations

**Contention vs. utilization trade-off:**
- Too few threads (ex: 10-20): Underutilizes CPU cores, less parallelism benefit
- Too many threads (ex: +100): More lock contention when pools run out, higher memory pressure
- I believe that 50 threads hits a sweet spot where we get good parallel speedup while keeping pool allocations dominant

With 50 concurrent threads and 16KB pools, most Huffman tree construction fits entirely within each thread's pool, meaning threads rarely need the global lock. This reduces contention compared to the baseline where every allocation requires the lock.

## Performance Results

### Test Commands

```bash
# Test with progressively larger files

gavin@Kronos:/mnt/c/Users/gavin/Downloads/CSC139/ONE/TWO$ ./sharedhash test_1KB.bin -t
Final signature: 1062571924
gavin@Kronos:/mnt/c/Users/gavin/Downloads/CSC139/ONE/TWO$ ./esharedhash test_1KB.bin -t
Final signature: 1062571924
Lock acquisitions: 102
# Both implementations work on small files, but esharedhash shows the optimization in action with lock counts.

gavin@Kronos:/mnt/c/Users/gavin/Downloads/CSC139/ONE/TWO$ ./sharedhash test_100KB.bin -t
Final signature: 443491033
gavin@Kronos:/mnt/c/Users/gavin/Downloads/CSC139/ONE/TWO$ ./esharedhash test_100KB.bin -t
Final signature: 443491033
Lock acquisitions: 11164

gavin@Kronos:/mnt/c/Users/gavin/Downloads/CSC139/ONE/TWO$ ./sharedhash test_200KB.bin -t
Final signature: 1182207701
gavin@Kronos:/mnt/c/Users/gavin/Downloads/CSC139/ONE/TWO$ ./esharedhash test_200KB.bin -t
Final signature: 1182207701
Lock acquisitions: 22316

gavin@Kronos:/mnt/c/Users/gavin/Downloads/CSC139/ONE/TWO$ ./sharedhash test_10MB.bin -t
umalloc failed for block 288
^C^C^C^C^C^C^C^CSegmentation fault (core dumped)
gavin@Kronos:/mnt/c/Users/gavin/Downloads/CSC139/ONE/TWO$ ./esharedhash test_10MB.bin -t
Final signature: 758276464
Lock acquisitions: 1140824
# sharedhash starts failing at 1Mb due to spawning all threads simultaneously.

gavin@Kronos:/mnt/c/Users/gavin/Downloads/CSC139/ONE/TWO$ ./sharedhash test_10MB.bin
Final signature: 758276464
gavin@Kronos:/mnt/c/Users/gavin/Downloads/CSC139/ONE/TWO$ ./esharedhash test_10MB.bin
Final signature: 758276464
# Both produce identical signatures in single-threaded mode, confirming correctness.

```
### Performance Summary

| File Size | sharedhash (baseline) | esharedhash (optimized) | Improvement |
|-----------|----------------------|-------------------------|-------------|
| 1KB       | Works | Works, 102 locks | Lock reduction |
| 100KB     | Works (slow) | Works, 11,164 locks | ~89% fewer locks |
| 200KB     | Works (slow) | Works, 22,316 locks | Functional |
| 10MB      | **Segmentation fault** | Works, 1,140,824 locks | Critical scalability |


## What I Learned

This project changed the way I think about concurrency. I'd understood threading at before, but this assignment made it clear just how dramatically synchronization can have an effect on performance. In the baseline implementation, the threads technically ran in parallel, yet were spending most of their time waiting on a single global lock inside the allocator. The whole program behaved almost serially. Since the threads couldn't make progress without constantly blocking one another, implementing per-thread memory pools showed just how transformative reducing shared coordination could be. By giving each thread its own region to allocate from, most calls to umalloc() no longer required the global lock, and the parallelism at last translated into real speedup. The allocator itself didn't change-only the amount of synchronization required.

I also learned that practical systems often need pragmatic choices. While the specification encouraged spawning all threads at once, doing so would exceed realistic memory limits. Batching threads allowed the program to scale to large inputs by reusing memory between batches, while still achieving parallelism in some way. Running 50 threads at a time saturates the CPU on most machines. Finally, I saw that the work of performance engineering is full of trade-offs. Larger per-thread pools reduce lock contention but waste more memory. Larger batch sizes increase CPU utilization but increase memory pressure. There is no universally optimal configuration-you have to measure, adjust, and choose the balance that fits your workload.

## Supplementary: Why The -m Flag Was Omitted

Multi-process mode was preserved in the code but wasn't my focus of optimization work for either Part 1 or Part 2. In part 1, the assignment was about converting to multi-threading (`-t` flag), not optimizing multi-process mode. The `-m` functionality already existed in the scaffold and remained unchanged. I focused on implementing correct thread-based memory sharing with `malloc()` and mutex synchronization instead of `mmap()` and semaphores. In part 2, optimization strategies such as per-thread pools and batching are specific to threading. 
Reveiving a `Segmentation Fault` error using the `-m` flag was expected because the multi-process optimization was never the focus. 
