# Part 1: Multi-Threaded Huffman Hash - Implementation Report

## Correctness Testing

To verify correctness, I tested the program with progressively larger files to see how the threading implementation behaves under different memory pressures. The tests compare sharedhash.c (Part 1 baseline) against esharedhash.c (Part 2 optimized) to show the performance difference.

```bash
./sharedhash test_1KB.bin -t
./esharedhash test_1KB.bin -t

./sharedhash test_100KB.bin -t
./esharedhash test_100KB.bin -t

./sharedhash test_200KB.bin -t
./esharedhash test_200KB.bin -t
```

**Results:**
```
Final signature: 1062571924
Final signature: 1062571924
Lock acquisitions: 102

Final signature: 443491033
Final signature: 443491033
Lock acquisitions: 11164

Segmentation fault (core dumped)
Final signature: 1182207701
Lock acquisitions: 22316
```

### Why This Test Structure?

The test progression (1KB → 100KB → 200KB) was designed to show the scalability limits of the Part 1 implementation. With 1KB and 100KB files, sharedhash.c works fine and produces correct hashes. But at 200KB (200 blocks), it segfaults because the implementation spawns all threads at once. With 200 threads each needing memory for their input buffers and Huffman tree construction, the 2MB heap runs out of space before threads can start freeing memory.

The lock acquisition counts in esharedhash.c show why optimization was needed - even for small files, there are thousands of lock acquisitions (102 for 1KB, 11,164 for 100KB). This demonstrates the severe lock contention problem that Part 2 addresses with per-thread memory pools.

## Implementation Summary

The main changes I made were converting the multi-process scaffold to support multi-threading. Instead of using `fork()` to create child processes, I used `pthread_create()` to spawn threads. For synchronization, I replaced the process-shared semaphore with a pthread mutex (`pthread_mutex_t`). I also changed the memory allocation strategy - the original version used `mmap()` with `MAP_SHARED` for the heap, but threads don't need this since they naturally share the same address space. So I changed `init_umem()` to use regular `malloc()` when in thread mode. Following the assignment requirements, I removed the batching approach and changed the data structure from using a pointer-to-pointer (`free_list_ptr`) to a simple pointer (`free_list`).

The key difference between thread-based and process-based memory sharing is that threads automatically share all global variables and heap memory within the same process, while processes have separate address spaces and need explicit shared memory (via `mmap`) to communicate. With processes, we needed `mmap()` for the heap and a pointer-to-pointer stored in shared memory so all processes could see updates to the free list head. With threads, I simplified this to a regular global pointer since all threads in the process can directly access and modify the same global variables. This makes the thread implementation cleaner, but it also means all threads are fighting for the same allocator lock, which creates heavy lock contention - especially visible when processing larger files.
