# Hash Hash Hash
``` shell
In this lab, we are making sure that a hash table implementation is thread safe. Particularly, we are making sure that the race conditions associated with inserting entries are safely handled and avoided, and on one implementation, ensuring that performance is not damaged by these precautions.
```

## Building
```shell
To build this, we run the `make` command. This will create the executable called "hash-table-tester" and needed binary files.
```

## Running
``` shell
To run this, we can run the command

./hash-table-tester -t x -s y

Where x is the number of threads we want to use, and y is the number of elements we would like to add to our hash table. As mentioned in the spec, the default number of elements to be added is 25,000 and the default number of threads is 4. 

The outputs of this command will be in the form

Generation: [seconds] usec
Hash table base: [seconds] usec
  - [# entries] missing
Hash table v1: [seconds] usec
  - [# entries] missing
Hash table v2: [seconds] usec
  - [# entries] missing

If successful, the output should always have 0 entries missing for base, v1, and v2. Additionally, performance should be as follows:

v1 > base (by an arbitrary amount)
v2 < base (by about t-1 times, where t = # threads inputted by user)
```

## First Implementation
```shell 
In the `hash_table_v1_add_entry` function, I added a single mutex to protect the entry into the list within the hash table struct. This mutex was placed around the call to insert the element into the list within the hash table and the check to its existence already, in which it was unlocked so that we can avoid the condition in which an element exists and the mutex is unlockable.

I created this mutex within the hash table definition itself so that it could be managed by the hash table and not created & destroyed on each individual call to the `hash_table_v1_add_entry` function, since that does not adequately address the race conditions.

Additionally, in the destructor of the hash table, I destroyed the mutex, accounting for the errno and returning that if necessary.

The addition of the lock around this critical section successfully protects against the race condition in which multiple threads can modify an entry of hash table, adding to the linked list within. Threads can now safely access the hash table overall one-at-a-time.

```

### Performance
```shell
Version 1 is a slower than the base version, which is a result of the addition of the lock. The purpose of the threads are to provide parallelism and allow multiple aspects of a process to happen at once. By adding the lock around the entirety of the list, I reduce parallelism by only granting one thread at a time access into the hash table entry. Thus, the runtime is signficantly slower.
 

With this implementation, when I run: 

./hash-table-tester -t 4 -s 10000
Generation: 8,326 usec
Hash table base: 9,105 usec
  - 0 missing
Hash table v1: 10,553 usec
  - 0 missing

Thus, we correct for the insertion issue, but decrease the overall performance (decrease meaning make it slower!), in this case, by around 1.15 times the base. 
```

## Second Implementation
```shell
In the `hash_table_v2_add_entry` function, I created an array of mutexes, one for each entry in the hash table, which is at most 4096. Each mutex corresponds to an index in the hash table, so mutex[1] protects, entry 1 in the table, mutex[2] protects entry 2, and so on. By doing this, we are able to lock a singular entry in the hash table instead of the entire thing, allowing for more parallelism and efficiency, while still maintaining the race condition fix that we created in v1. Threads can work in parallel so long as they are working on different entries. Because of this, we don't need to add any additional locks anywhere else. This is the biggest/only part in which a race condition would occur, and other work on an entry is protected by the mutex.

I placed the mutex around the call to actually insert again, but this time taking into account the `bernstein hash` function to access the index for both the mutex and the entry specifically. Like before, I unlocked the mutex within the if statement checking for if the element already exists to avoid the edge case where the mutex is not unlocked. This is done after updating the value. Placing the lock right around the call optimizes efficiency but also takes into account the critical section and delicately manages it. 

Similar to before, I created the mutex array in the hash table constructor by declaring the array of size HASH_TABLE_CAPACITY for an array of pthread_mutex_t.

I simply added the code to the for loop that already existed to initialize the entries of the hash table to initialize each mutex to null to begin with.

Similarly, I added a line of code to the destructor's for loop to properly destroy each mutex i. 

For each call to a pthread function, I collected and returned an errno if it was nonzero.

By doing this, I was able to protect each entry in the hash table from race conditions and handle the creation and destruction of the mutex to avoid memory leaks. This overall increased the parallelism of the program; threads are now able to work at the same time such that they are working on different indexes of the hash table. 

```
 
### Performance
```shell
Version 2 is faster than the base version, which is a result of the addition of the mutexes around each individual entry in the hash table rather than the hash table overall. Parallel threads are able to work in sync to process the instructions of the program but on separate entries of the program. Each thread can grab a lock for a specific entry and insert elements there as needed, before releasing the lock to another thread and continuting. when I run

./hash-table-tester -t 4 -s 10000

I get: 

Generation: 7,542 usec
Hash table base: 12,066 usec
  - 0 missing
Hash table v1: 12,744 usec
  - 0 missing
Hash table v2: 1,954 usec
  - 0 missing

As we can see above, the runtime for v2 is monumentally faster than both the base and v1's implementation. This is an indicator of the success of our locking strategy, where each entry's protection results in increased parallelization. Specifically, the runtime for v2 is 6 times faster than the base. The target here is about t-1 times faster, which in this case is 3. This output well-exceeds that!
```

## Cleaning up
```shell
To clean up, we run the `make clean` function. This will remove all binary and executable files.
```