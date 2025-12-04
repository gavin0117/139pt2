# Part 2 Performance Report: Per-Thread Memory Pools

## Optimization Strategy

I implemented **Strategy A: Per-Thread Memory Pools** from the Part 2 specification. This approach eliminates the global allocator lock bottleneck by giving each thread its own dedicated memory region.

### Why This Strategy?

The baseline `sharedhash.c` implementation uses a single global mutex to protect the entire memory allocator. With 20 concurrent threads each performing hundreds of allocations to build Huffman trees, this creates severe lock contention. Threads spend most of their time waiting for the lock rather than doing useful work.

Per-thread pools eliminate this contention by allowing each thread to allocate from its own private region without any synchronization overhead.

## Implementation Details

### Key Components

1. **Thread-Local Storage**: Each thread maintains its own pool using `__thread` variables:
   - `pool_start`: Beginning of the thread's memory region
   - `pool_current`: Bump allocator pointer (increments on each allocation)
   - `pool_size`: Total size of the thread's pool (32KB)

2. **Pool Allocation**: Reserved 640KB at the beginning of the heap for 20 pools (matching BATCH_SIZE)

3. **Pool Recycling**: Since we process threads in batches of 20, we reset the pool counter between batches. This allows unlimited file sizes while only using 20 pools.

4. **Fast Path**: `umalloc()` first tries the thread pool with a simple bump allocator (no locking). Only falls back to the global allocator if the pool is exhausted.

5. **Smart Freeing**: `ufree()` detects whether a pointer came from a thread pool or global allocator. Pool allocations are not freed (bump allocators don't support freeing), while global allocations are freed normally.

## Performance Results

### Test Environment
- Platform: WSL (Linux)
- Heap Size: 2MB
- Thread Pool Size: 32KB per thread
- Max Concurrent Threads: 20 (BATCH_SIZE)

### Timing Comparison

| File Size | sharedhash (baseline) | esharedhash (optimized) | Speedup |
|-----------|----------------------|-------------------------|---------|
| 1MB       | 7.225s              | 0.277s                 | **26.1x** |
| 5MB       | 43.300s             | 1.278s                 | **33.9x** |

### Lock Acquisition Counts

| File Size | Blocks | sharedhash Lock Count | esharedhash Lock Count | Reduction |
|-----------|--------|----------------------|------------------------|-----------|
| 1MB       | 1,024  | ~1,000,000+         | 2,048                 | **99.8%** |
| 5MB       | 5,120  | ~5,000,000+         | 10,240                | **99.8%** |

### Analysis

The lock count pattern is revealing:
- In `esharedhash`, lock count ≈ 2 × number of blocks
- This matches our design: each thread acquires the lock once for its input buffer allocation and once when the pool runs out near the end
- The vast majority of Huffman tree allocations happen from thread pools without any locking

## What I Learned

### 1. Lock Contention is a Performance Killer

The baseline implementation was **correct** but **inefficient**. Even with 20-way parallelism, threads spent most of their time blocked on the allocator lock. The speedup of ~30x demonstrates how much time was wasted on synchronization rather than computation.

### 2. Concurrent vs. Parallel Execution

Having multiple threads doesn't guarantee parallel speedup. In the baseline:
- Threads ran **concurrently** (switching between them)
- But NOT truly **parallel** (making progress simultaneously)

The global lock serialized execution at the allocator level, forcing threads to take turns.

### 3. Trade-offs in Allocator Design

Per-thread pools achieve dramatic speedup but have downsides:
- **Memory waste**: If threads allocate unevenly, some pools sit mostly empty
- **No freeing**: Bump allocators can't recycle memory within a thread's execution
- **Fixed pool size**: Chose 32KB through experimentation (too small = frequent fallback, too large = wasted space)

### 4. Batched Threading Design

The initial implementation created all 1,024 threads upfront, which exhausted the 2MB heap. The batched approach (BATCH_SIZE=20) was critical:
- Limits concurrent memory usage
- Enables pool recycling between batches
- Allows scaling to arbitrarily large files

### 5. Pool Size Tuning

Through experimentation:
- 2KB pools: Too small, frequent fallback to global allocator
- 8KB pools: Better, but still high lock count without pool recycling
- 32KB pools with recycling: Sweet spot - most blocks complete entirely from pool

## Conclusion

Per-thread memory pools transformed the allocator from a severe bottleneck into a negligible overhead. The key insight is that **reducing lock contention** is more valuable than sophisticated allocation algorithms when parallelism is the goal.

This experiment demonstrates the difference between **concurrent** programs (correct but slow) and **parallel** programs (truly utilizing multiple cores). The ~30x speedup shows that allocation overhead, not computation, was the limiting factor in the baseline implementation.
