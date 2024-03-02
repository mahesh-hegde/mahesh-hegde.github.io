---
title: "OpenMP and rand() function - A small story"
type: "post"
date: 2023-01-07T18:35:13+05:30
draft: false
tags: [C, Performance]
---

I have a parallel programming subject this semester, and as usual, there's also a lab in which we try and execute small programs (mostly using OpenMP).

First program is to find the value of `Pi` using Monte carlo methods. We have to then measure the performance with different number of threads, and tabulate it.

Usually, when we add threads using `pragma omp parallel for` directive, we expect the program to take less time. Here's an implementation that everyone wrote:

```c
#pragma omp parallel for
for (int i = 0; i < nIter; i++) {
	x = (double)rand() / RAND_MAX;
	y = (double)rand() / RAND_MAX;
	if ((x*x + y*y) <= 1) count++;
}
return 4 * (double)count/nIter;
```

But surprisingly, this doesn't give the results you would expect. In fact, the run time strictly increases with number of threads.

![Counter-intuitive results with naive openmp pragma](/images/omp_rand_critical_section/Screenshot_Before.png)

I was surprised, but seeing that everyone got the same results, I too first ignored the problem. But it was kind of a surprising result.

However, later I realized the problem. `rand()` is called from all threads. Each subsequent output of `rand()` is different so apparently it stores some global state. And usually that means locking.

Turns out my suspicion was true. I looked around in the [glibc sources](https://sourceware.org/git/?p=glibc.git&a=search&h=HEAD&st=grep&s=rand%28%29). There's a locking call in [random.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=stdlib/random.c;hb=ae612c45efb5e34713859a5facf92368307efb6e) and this comment confirms it.

```
194 /* POSIX.1c requires that there is mutual exclusion for the `rand' and
195    `srand' functions to prevent concurrent calls from modifying common
196    data.  */
```

So rand() is obtaining a mutex, which is negating any benefit we get from OpenMP pragmas.

Solution? This is what I arrived at with some fiddling. Basically using a local, reentrant version of `rand` - called `rand_r` and using a private variable to store the random state.

By doing this, each thread has a local state for random number generation. Thus locking will not be needed.

```c
#pragma omp parallel
{
int localCount = 0;
// This is the seed value for rand_r function.
// (should actually use something better here than clock)
unsigned int randomState = clock();
#pragma omp for
for (int i = 0; i < nIter; i++) {
	x = (double)rand_r(&randomState) / RAND_MAX;
	y = (double)rand_r(&randomState) / RAND_MAX;
	if ((x*x + y*y) <= 1) localCount++;
}
#pragma omp critical
count += localCount;
}
return 4 * (double)count/nIter;
```

This fixes the speed problem. But there's still a limit to parallelism depending on your machine's specs, and there may be further bottlenecks in the program as well.

But for the purpose of the lab, I get a decent graph which shows an increase in speed :P.

![After eliminating critical section](/images/omp_rand_critical_section/Screenshot_After.png)

---

Update (July 02): This [reddit thread](https://old.reddit.com/r/C_Programming/comments/14oib3m/openmp_and_rand_function_a_small_story/) is very informative discussion on this post. Especially [this comment by u/skeeto](https://old.reddit.com/r/C_Programming/comments/14oib3m/openmp_and_rand_function_a_small_story/jqdncfj/) recommends to use a single global seed than `clock()` in every thread.

