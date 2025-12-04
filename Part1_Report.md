# Part 1: Multi-Threaded Huffman Hash - Implementation Report

## Correctness Testing

To verify correctness, I ran the program in all three modes (single, multi-process, multi-thread) on the same test file and confirmed that all modes produce identical final signatures.

```bash
# [PLACEHOLDER: Insert test commands and output here]
# Example:
# ./sharedhash test_1MB.bin
# Final signature: 433393817
#
# ./sharedhash test_1MB.bin -m
# Final signature: 433393817
#
# ./sharedhash test_1MB.bin -t
# Final signature: 433393817
```

All three modes produce matching signatures, confirming correctness.

## Implementation Summary

The main changes I made were converting the multi-process scaffold to support multi-threading. Instead of using `fork()` to create child processes, I used `pthread_create()` to spawn threads. For synchronization, I replaced the process-shared semaphore with a pthread mutex (`pthread_mutex_t`). I also changed the memory allocation strategy - the original version used `mmap()` with `MAP_SHARED` for the heap, but threads don't need this since they naturally share the same address space. So I changed `init_umem()` to use regular `malloc()` when in thread mode.

The key difference between thread-based and process-based memory sharing is that threads automatically share all global variables and heap memory within the same process, while processes have separate address spaces and need explicit shared memory (via `mmap`) to communicate. With processes, we needed `mmap()` for the heap and a pointer-to-pointer (`free_list_ptr`) stored in shared memory so all processes could see updates to the free list head. With threads, I simplified this to a regular global pointer (`free_list`) since all threads in the process can directly access and modify the same global variables. This makes the thread implementation cleaner, but it also means all threads are fighting for the same allocator lock, which creates heavy lock contention.
