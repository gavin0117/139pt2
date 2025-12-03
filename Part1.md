Part 1: Converting the Shared 
Memory Hasher to Threads 
In the provided sharedhash.c program, each 1KB block of input is processed by a 
separate fork()ed process. These child processes share access to a global heap 
allocator through mmap()-based shared memory, coordinating access with a 
process-shared semaphore. This model is correct but costly. Every fork() call 
duplicates address space structures, incurs kernel overhead, and results in 
heavyweight synchronization. In this part, you will convert the program from a 
multi-process architecture to a multi-threaded one using the pthread library. The goal 
is to preserve correctness while eliminating the unnecessary complexity and overhead 
of inter-process communication. 
1. Conceptual Shift: Processes vs. Threads 
Before making changes, it’s essential to understand how threads differ from processes. 
Processes have independent address spaces; after fork(), memory is only shared 
when explicitly mapped. Threads, on the other hand, automatically share their entire 
address space, including the heap, globals, and static variables. This means: 
● You will remove all mmap() calls that were used solely for sharing memory 
between processes. 
● The allocator will naturally operate on shared memory—no special setup is 
needed for inter-thread visibility. 
● The semaphore used for process coordination will be replaced by a pthread 
mutex. 
The practical effect is a much simpler setup: instead of creating shared memory 
manually, you will rely on the implicit shared address space that all threads inhabit. 
// Old (process-shared) 
sem_t* mLock; 
sem_init(mLock, 1, 1);  // pshared = 1 
// New (thread-shared) 
pthread_mutex_t mLock = PTHREAD_MUTEX_INITIALIZER; 
2. Step-by-Step Conversion Plan 
Follow this sequence carefully. Each step has been designed to minimize confusion 
and prevent cascading bugs. 
Step 1: Replace Process Creation with Threads 
In the original code, run_multi() uses fork() to spawn a new process for each 
block and pipes for communication. In your thread-based version, you will replace this 
with pthread_create() calls. Each thread will handle one block of data and write its 
hash result into a shared array. 
// Thread argument structure 
typedef struct { 
int block_id; 
unsigned char *block_buf; 
size_t block_len; 
unsigned long *results; 
} thread_arg_t; 
// Worker thread function 
void *worker_thread(void *arg) { 
thread_arg_t *targ = (thread_arg_t *)arg; 
unsigned long h = process_block(targ->block_buf, 
targ->block_len); 
ufree(targ->block_buf); 
targ->results[targ->block_id] = h; 
return NULL; 
} 
Each thread computes its hash independently. The parent thread (the main program) 
will later join all threads and combine their results into a final hash signature. 
Step 2: Remove Pipes and Replace with Shared Results 
In the multi-process version, pipes were used to collect results from children. With 
threads, you can replace all pipe logic with a simple array of unsigned long results. 
// Global or stack array 
unsigned long results[1024]; 
// After joining threads 
unsigned long final_hash = 0; 
for (int i = 0; i < num_blocks; i++) { 
print_intermediate(i, results[i], 0); 
final_hash = (final_hash + results[i]) % LARGE_PRIME; 
} 
print_final(final_hash); 
Since all threads share memory, no explicit inter-thread communication mechanism 
(like pipes) is needed. Each thread writes its result directly to its own index in the array. 
Step 3: Replace Semaphore with a Mutex 
Your allocator currently uses sem_wait() and sem_post() to protect critical 
sections. Replace them with pthread_mutex_lock() and 
pthread_mutex_unlock(). Also, remove sem_init() from the setup code and 
replace it with a static mutex declaration. 
// Replace: 
sem_wait(mLock); 
... 
sem_post(mLock); 
// With: 
pthread_mutex_lock(&mLock); 
... 
pthread_mutex_unlock(&mLock); 
For single-threaded mode, you can keep the same use_multiprocess flag (rename it 
to use_multithread if desired). Conditional locking ensures the single-threaded path 
avoids unnecessary synchronization overhead. 
Step 4: Replace Shared Memory Setup with Local Allocations 
In init_umem(), remove all mmap() calls and replace them with regular malloc(). 
Threads naturally share global variables, so there’s no need for shared memory 
mappings. 
static node_t *free_list = NULL; 
void *init_umem(void) { 
void *base = malloc(UMEM_SIZE); 
if (!base) { 
perror("malloc"); 
exit(1); 
} 
free_list = (node_t *)base; 
free_list->size = UMEM_SIZE - sizeof(node_t); 
free_list->next = NULL; 
return base; 
} 
By converting free_list_ptr (a pointer-to-pointer) to a regular pointer, you eliminate 
unnecessary indirection. Threaded programs inherently share the same memory space, 
so a global pointer suffices. 
Step 5: Convert run_multi() into run_threads() 
The main driver for the multi-threaded version should closely mirror the structure of the 
process-based version. You’ll still read the file block-by-block, allocate memory for 
each, and dispatch workers. The key differences are that you now: 
● Call pthread_create() instead of fork() 
● Pass thread arguments instead of writing to pipes 
● Use pthread_join() instead of waitpid() 
Step 6: Update main() 
Your main() function should select between single-threaded and multi-threaded 
modes based on the command-line flag -t (for “threaded”). For compatibility, you can 
keep the existing -m for processes, although the autograder will focus on the threaded 
mode. 
int main(int argc, char *argv[]) { 
if (argc < 2) { 
fprintf(stderr, "Usage: %s  [-m|-t]\n", argv[0]); 
return 1; 
} 
const char *filename = argv[1]; 
use_multiprocess = (argc >= 3 && strcmp(argv[2], "-m") == 0); 
int use_multithread = (argc >= 3 && strcmp(argv[2], "-t") == 0); 
init_umem(); 
if (use_multithread) 
return run_threads(filename); 
else if (use_multiprocess) 
return run_multi(filename); 
else 
return run_single(filename); 
} 
3. Testing Your Threaded Implementation 
Once your thread-based version compiles, begin testing gradually: 
1. Run in single-threaded mode and verify it matches the baseline output. 
2. Run with -t using small files (1–10KB). 
3. Increase input size to confirm stability under higher concurrency. 
4. Use valgrind to detect memory leaks or invalid access patterns. 
If your threaded version produces different hashes than the process version, possible 
causes include: 
● Race conditions due to missing or incorrect locks in the allocator 
● Incorrect indexing of the results array 
● Reusing the same buffer pointer for multiple threads 
● Not joining all threads before aggregating results 
4. Expected Behavior and Performance 
The threaded version should behave identically to the process-based one from a 
correctness perspective: the same input file should yield the same final hash signature. 
However, you should notice a difference in performance and resource usage: 
● Lower overhead: Threads are lighter than processes and share code, data, 
and file descriptors. 
● No pipes or copies: Data sharing is direct, not via interprocess 
communication. 
● Improved responsiveness: Fewer system calls and context switches. 
● Limited speedup: Still limited by the coarse-grained allocator lock. 
Run a few timing tests to confirm the expected behavior: 
time ./sharedhash test_1MB.bin 
time ./sharedhash test_1MB.bin -m 
time ./sharedhash test_1MB.bin -t 
You may observe that the threaded version is slightly faster than the process version 
but still far from ideal scaling. That’s expected—the single global mutex in the allocator 
still forces threads to wait for one another. In the next part, you’ll redesign the allocator 
to reduce this contention and measure actual speedup. 
5. Common Mistakes and Debugging Advice 
The following issues are common in this conversion: 
● Stack variable lifetime: Do not pass the address of a stack variable (like 
thread_arg_t arg;) to pthread_create(). Use an array of argument 
structs so that each thread gets a stable address. 
● Forgetting to join threads: Threads that aren’t joined can cause undefined 
behavior or missing results. 
● Mixing semaphores and mutexes: After conversion, ensure there are no 
lingering sem_* calls. 
● Not freeing memory: Verify that ufree() is called for each umalloc() 
allocation to avoid leaks. 
● Uninitialized globals: Remember that the mutex must be initialized statically 
or with pthread_mutex_init(). 
6. Deliverables for Part 1 
Your submission for Part 1 should include: 
● A working sharedhash.c that supports -t for threaded execution. 
● Output from correctness tests confirming that single, process, and thread 
modes produce identical final signatures. 
● A brief explanation (1–2 paragraphs) describing the main changes you made 
and how thread-based memory sharing differs from process-based shared 
memory. 
● All writeup must be submitted in a markdown file (.md) 
7. Grading Criteria 
Points for Part 1 are awarded based on: 
● Correctness: identical outputs across all modes 
● Proper use of pthread APIs and mutexes 
● Memory safety 
● Code readability and comments 
● Successful test results and explanation 
Passing this stage confirms that you understand the fundamental differences between 
process and thread concurrency and can implement a correct, shared-memory 
allocator in both environments. The next part focuses on improving performance by 
reducing lock contention and introducing finer-grained synchronization. 
