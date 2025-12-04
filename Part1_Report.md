# Part 1: Multi-Threaded Huffman Hash - Implementation Report FINAL

## Correctness Testing

I tested the program with progressively larger files to see how the threading implementation will behaves under different memory pressures. The tests compare sharedhash.c (Part 1 baseline) against esharedhash.c (Part 2 optimized) to show the performance difference.

```bash
gavin@Kronos:/mnt/c/Users/gavin/Downloads/CSC139/ONE/TWO$ ./sharedhash test_1KB.bin
Final signature: 1062571924
gavin@Kronos:/mnt/c/Users/gavin/Downloads/CSC139/ONE/TWO$ ./sharedhash test_1KB.bin -t
Final signature: 1062571924

gavin@Kronos:/mnt/c/Users/gavin/Downloads/CSC139/ONE/TWO$ ./sharedhash test_100KB.bin
Final signature: 443491033
gavin@Kronos:/mnt/c/Users/gavin/Downloads/CSC139/ONE/TWO$ ./sharedhash test_100KB.bin -t
Final signature: 443491033

gavin@Kronos:/mnt/c/Users/gavin/Downloads/CSC139/ONE/TWO$ ./sharedhash test_200KB.bin
Final signature: 1182207701
gavin@Kronos:/mnt/c/Users/gavin/Downloads/CSC139/ONE/TWO$ ./sharedhash test_200KB.bin -t
Segmentation fault (core dumped)
```

### Why This Route? 

The progression (1KB to 100KB to 200KB) was chosen to show the limits of the Part 1 implementation. With 1KB and 100KB files, sharedhash.c works fine and produces correct hashes. But at 200KB, or 200 blocks, it segfaults because the implementation spawns all threads at once. With 200 threads each needing memory for their input buffers and Huffman tree construction, the 2MB heap runs out of space before threads can start freeing memory.

The lock acquisition counts in esharedhash.c show why optimization was needed, there are thousands of lock acquisitions (102 for 1KB, 11,164 for 100KB). This shows the severe lock contention problem that Part 2 addresses with per-thread memory pools.

## Overall and Final Summary

The main changes I made were converting the multi-process scaffold to support multi-threading. Instead of using `fork()` to create child processes, instead, i used  `pthread_create()` to spawn threads. For synchronization, I replaced the process-shared semaphore with a pthread mutex (`pthread_mutex_t`). I also changed the memory allocation strategy. For reference, the code given to use used `mmap()` with `MAP_SHARED` for the heap, but threads don't need this since they naturally share the same address space. So I changed `init_umem()` to use regular `malloc()` when in thread mode. I also decided to remove the batching approach and changed the data structure from using a pointer-to-pointer (`free_list_ptr`) to a simple pointer (`free_list`).
The key difference between thread-based and process-based memory sharing is that threads automatically share all global variables and heap memory within the same process. While it is true that processes have separate address spaces and need explicit shared memory via `mmap` to communicate. With processes, we needed `mmap()` for the heap and a pointer-to-pointer stored in shared memory so all processes could see updates to the free list head. Additionally, with threads, I simplified this to a regular global pointer since all threads in the process can directly access and modify the same global variables. This makes the thread implementation cleaner, but it also means all threads are fighting for the same allocator lock, which creates heavy lock contention, this is especially visible when processing larger files. In particular, files over 1Mb in size. Overall, I had a difficult time choosing the decisions that were ultimately made in this part but it ultimately came together to form the first half of this project.

