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
In the `hash_table_v1_add_entry` function, I added a single mutex to protect the entry into the list within the hash table struct. This mutex was placed around the call to insert the element into the list within the hash table and after the check to its existence already:

_list_entry = calloc(1, sizeof(struct list_entry));_
	_list_entry->key = key;_
	_list_entry->value = value;_
	_SLIST_INSERT_HEAD(list_head, list_entry, pointers);_

I created this mutex within the hash table definition itself so that it could be managed by the hash table and not created & destroyed on each individual call to the `hash_table_v1_add_entry` function, since that does not adequately address the race conditions:

_struct hash_table_v1 {_
	_struct hash_table_entry entries[HASH_TABLE_CAPACITY];_
	_pthread_mutex_t mutex;_ //added the mutex here!
_};_

Additionally, in the destructor of the hash table, I destroyed the mutex:

_errno = pthread_mutex_destroy(&hash_table->mutex);_
	_if(errno){ exit(errno);}_
	_free(hash_table);_

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
In the `hash_table_v2_add_entry` function, I created an array of mutexes, one for each entry in the hash table, which is at most 4096. By doing this, we are able to lock a singular entry instead of the entire list, allowing for more parallelism and efficiency, while still maintaining the race

```
 
### Performance
```shell
TODO how to run and results
```

TODO more results, speedup measurement, and analysis on v2

## Cleaning up
```shell
To clean up, we run the `make clean` function. This will remove all binary and executable files.
```