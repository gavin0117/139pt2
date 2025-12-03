Part 2: Exploring Concurrency 
and Optimizing the Allocator 
In Part 1, you converted the shared memory hasher from a process-based program to 
a thread-based one. Your solution was correct and functional, but its performance 
remained poor. Nearly every call to umalloc() and ufree() required a global mutex 
lock, forcing all threads to take turns accessing the allocator. This means that even 
though your threads ran concurrently, they could not make real progress in parallel. In 
this final part, you will investigate ways to reduce this bottleneck by experimenting with 
allocator-level optimizations. The goal is to explore trade-offs in concurrency—not 
necessarily to build the fastest allocator, but to reason about why it performs the way it 
does. 
1. Starting Point and File Naming 
Begin with your completed sharedhash.c from Part 1. Create a copy named 
esharedhash.c. This will be your experimental version. You will submit both files: 
● sharedhash.c — your correct, fully working threaded implementation from 
Part 1 
● esharedhash.c — your modified experimental version for this part 
This separation allows you to try aggressive changes without breaking your working 
baseline. You should be able to recompile and compare both versions side by side: 
gcc -pthread -o sharedhash sharedhash.c 
gcc -pthread -o esharedhash esharedhash.c 
Then test them with: 
time ./sharedhash test_1MB.bin -t 
time ./esharedhash test_1MB.bin -t 
Your goal is not just to make esharedhash faster, but to understand why it behaves 
differently. 
2. The Bottleneck: Coarse-Grained Locking 
In the baseline implementation, the allocator uses one global lock (pthread_mutex_t 
mLock) to protect the entire free list. This ensures safety but serializes every allocation 
and free operation. With 10 threads performing hundreds of allocations each, the lock 
becomes a choke point. When one thread calls umalloc(), all other threads block, 
even if they are allocating memory from unrelated parts of the heap. 
You can confirm this behavior by adding an atomic counter around the lock calls: 
#include <stdatomic.h> 
atomic_int lock_count = 0; 
void *umalloc(size_t size) { 
atomic_fetch_add(&lock_count, 1); 
pthread_mutex_lock(&mLock); 
void *ptr = _umalloc(size); 
pthread_mutex_unlock(&mLock); 
return ptr; 
} 
At the end of main(), print the number of lock acquisitions and compare the counts 
between sharedhash and esharedhash. If your changes work, esharedhash 
should show significantly fewer lock acquisitions per run. 
3. Objective of Part 2 
Your task is to redesign or modify the allocator to reduce contention while preserving 
correctness. There are multiple directions you can explore. You are free to choose your 
approach, but you must explain and justify it in your write-up. The only hard 
requirements are: 
● Your allocator must remain memory safe—no corruption, no invalid reads or 
writes. 
● Your program must still produce the same final hash signature as before. 
● Your code must compile cleanly with -pthread and -Wall flags. 
● You must demonstrate a measurable difference in performance and explain 
it. 
The following sections describe several strategies you can try. Choose one and explore 
it in depth, or experiment with multiple if you have time. 
4. Strategy A: Per-Thread Memory Pools 
The simplest and most effective optimization is to eliminate the global lock by giving 
each thread its own allocator region. Since threads do not interfere with one another, 
most umalloc() and ufree() calls can proceed without synchronization. 
Concept 
Divide the shared heap into N equal regions, one per thread. Each thread maintains its 
own private free list or bump pointer within its region. Allocations and frees within that 
region require no locking. Only when a thread’s region runs out of space should it 
acquire the global lock and request additional memory. 
Implementation Sketch 
__thread char *pool_start = NULL; 
__thread char *pool_current = NULL; 
__thread size_t pool_size = 0; 
void thread_pool_init(int tid, int num_threads) { 
size_t chunk = UMEM_SIZE / num_threads; 
pool_start = (char *)global_heap + (tid * chunk); 
pool_current = pool_start; 
pool_size = chunk; 
} 
void *umalloc_fast(size_t size) { 
size = ALIGN(size); 
if (pool_current + size < pool_start + pool_size) { 
void *ptr = pool_current; 
pool_current += size; 
return ptr; 
} 
pthread_mutex_lock(&mLock); 
void *ptr = _umalloc(size); 
pthread_mutex_unlock(&mLock); 
return ptr; 
} 
This design removes the lock for most allocations, keeping the global lock as a 
fallback. You can ignore ufree() or make it a no-op for this experiment (acceptable, 
since the program exits soon after computation). Be sure to initialize the per-thread 
pool inside your thread’s start routine. 
Trade-offs 
● Very fast for workloads with independent allocations. 
● Wastes memory if threads allocate unevenly. 
● May require tuning pool sizes for different file sizes. 
5. Strategy B: Fine-Grained Locking 
Another approach is to keep a single shared heap but reduce contention by locking 
smaller regions independently. Instead of one lock for the entire free list, maintain 
multiple locks—one per size class or heap segment. Threads working with small 
allocations won’t block those working with large ones. 
#define NUM_CLASSES 8 
pthread_mutex_t locks[NUM_CLASSES]; 
node_t *free_lists[NUM_CLASSES]; 
int size_class(size_t size) { 
if (size <= 32) return 0; 
if (size <= 64) return 1; 
if (size <= 128) return 2; 
... 
return NUM_CLASSES - 1; 
} 
void *umalloc(size_t size) { 
int c = size_class(size); 
pthread_mutex_lock(&locks[c]); 
void *ptr = allocate_from_list(free_lists[c], size); 
pthread_mutex_unlock(&locks[c]); 
return ptr; 
} 
This design improves concurrency but increases complexity. Each free list must be 
managed separately, and coalescing becomes non-trivial when adjacent blocks belong 
to different size classes. 
Trade-offs 
● Better balance between concurrency and memory reuse. 
● More complex to debug; coalescing logic can become inconsistent. 
● Risk of internal fragmentation due to fixed-size bins. 
6. Strategy C: Opportunistic Locking and Batching 
You can also optimize without restructuring the heap. Instead of locking on every 
allocation, you can attempt trylock first, deferring allocations when the lock is busy 
or batching multiple allocations under a single lock hold. 
void *umalloc(size_t size) { 
if (pthread_mutex_trylock(&mLock) != 0) { 
// Lock busy, perform local caching or yield 
sched_yield(); 
pthread_mutex_lock(&mLock); 
} 
void *ptr = _umalloc(size); 
pthread_mutex_unlock(&mLock); 
return ptr; 
} 
Alternatively, you could hold the lock for several small allocations at once: 
void *alloc_many(size_t size, int count) { 
pthread_mutex_lock(&mLock); 
void *ptrs[count]; 
for (int i = 0; i < count; i++) 
ptrs[i] = _umalloc(size); 
pthread_mutex_unlock(&mLock); 
return ptrs[0]; 
} 
These techniques are heuristic—they trade fairness for throughput and may help on 
workloads with frequent small allocations. 
7. Measuring and Comparing Performance 
Once you have implemented an optimization, run both sharedhash and 
esharedhash on increasingly large inputs and compare the results. 
time ./sharedhash test_1MB.bin -t 
time ./esharedhash test_1MB.bin -t 
Then, try scaling the number of threads by increasing the number of blocks (larger files 
will automatically create more threads). You should see a measurable 
improvement—perhaps modest (2–3×) for fine-grained locking, or dramatic (5–8×) for 
per-thread pools. You can instrument your allocator to print lock acquisition counts and 
correlate them with performance differences. 
8. Guidelines for Experimentation 
Treat this part as an open-ended experiment. You are not required to reach a specific 
speedup, but you must reason about the design you chose and demonstrate that you 
understand the underlying trade-offs. Follow these guidelines: 
● Do not optimize at the expense of correctness. The hash output must remain 
identical. 
● Always re-run the single-threaded baseline before claiming a speedup. 
● Use small test cases to verify correctness, then large ones to observe 
performance effects. 
● Document any unexpected results—some optimizations can decrease 
performance due to cache contention or allocator overhead. 
● Use timing data and observations, not guesswork, in your write-up. 
9. Deliverables for Part 2 
You must submit the following: 
● Your modified esharedhash.c source file. 
● A short performance report (markdown is required) including: 
○ Which optimization strategy you implemented and why 
○ Timing data comparing sharedhash vs. esharedhash 
○ A short discussion of what you learned about concurrency and 
performance 
10. Grading Criteria 
● Correctness: output must match baseline exactly. 
● Performance Analysis: shows measurable difference and explains it. 
● Design Rationale: describes reasoning for chosen strategy. 
● Code Clarity and Documentation: clean, well-commented code. 
11. Closing Thoughts 
Part 2 represents the transition from building correct concurrent programs to designing 
efficient ones. There is no single best solution; each approach involves a balance 
between simplicity, safety, and scalability. By the time you finish, you should 
understand not just how to synchronize access to shared memory, but why 
performance often lags behind theoretical parallelism. This insight—the difference 
between concurrent and truly parallel execution—is the most valuable lesson in this 
exercise. 
Portfolio Use 
This assignment grants you permission to include your completed work in your 
professional portfolio. You may present your final version, with proper attribution, as an 
example of your ability to design and implement concurrent systems. Treat this project 
as a chance to produce something you’ll be proud to share, well-structured, functional, 
and thoughtfully documented. 
