---
title: Parallel Implementation of Prefix Sum
categories: [ programming ]
tags: [ cpp, openmp, boost ]
last_modified_at: 2023-03-02-T12:00:00-05:00
---

For a while, I have been invested in learning how to parallelize certain algorithms.
Parallelizing algorithms and code in general can be tricky. I have discovered that it is quite tricky.
It is surprisingly easy to create race conditions, and race conditions could go unnoticed for a long
time or deliver unreliable outputs.

## Prefix Sum - Serial

I was grinding LeetCode questions and Project Euler questions when I came across a problem that required
the use of a prefix sum. A prefix sum looks like the following example:

```bash
a = [ a0, a1, a2, a3 ]
p = [ a0, a0 + a1, a0 + a1 + a2 ]

p_n = a0 + a1 + ... + an
# OR
p_n = p_n-1 + an
```

Prefix sum is the running total of all numbers that occurred before it. It carries a dependency on the
previous elements being calculated before future values can be calculated. Here is a serial implementation.
Timing the program with an array of size 100'000'000, it only takes around a second to complete. In fact,
generating the random array took about 7x longer to populate.

```cpp
#include <vector>
#include <random>
#include <boost/timer/timer.hpp>

using boost::timer::auto_cpu_timer;

template <typename T>
std::vector<T> prefix_sum_serial(const std::vector<T>& input) {
    auto_cpu_timer t;

    if (input.size() == 0) {
        return {};
    }

    std::vector<T> result(input.size());
    result[0] = input[0];
    for (int i = 1; i < input.size(); ++i) {
        result[i] = result[i - 1] + input[i];
    }
    return result;
}

template <typename T>
std::vector<T> generate_random_array(const T lo, const T hi, const std::size_t size) {
    auto_cpu_timer t;

    std::random_device random_device;
    std::mt19937 random_engine(random_device());
    std::uniform_int_distribution<T> distribution(lo, hi);

    std::vector<T> numbers(size);

    for (size_t i = 0; i < numbers.size(); ++i)
    {
        numbers[i] = distribution(random_engine);
    }

    return numbers;
}

int main() {
    auto_cpu_timer t;

    const std::uint32_t lo(0);
    const std::uint32_t hi(999);
    const std::uint32_t size(100'000'000);

    const std::vector<std::uint32_t> input_array{ generate_random_array(lo, hi, size) };
    const std::vector<std::uint32_t> serial{ prefix_sum_serial(input_array) };
}
```

The timer shows that the `prefix_sum_serial` function finished executing in around 1 second, whereas
the `main` function took around 7 seconds to complete. Generating the array of random elements took around
4.7 seconds.

```bash
❯ /Users/kylehurd/Workplace/tmp/build/tmp
 4.705747s wall, 4.560000s user + 0.130000s system = 4.690000s CPU (99.7%)
 1.107477s wall, 1.030000s user + 0.080000s system = 1.110000s CPU (100.2%)
 6.956303s wall, 6.720000s user + 0.230000s system = 6.950000s CPU (99.9%)
```

## Generate Array of Random Numbers - Parallel

Given this information, parallelizing the prefix sum right now would not yield a large improvement due to the
`generate_random_array` function taking the majority of the runtime. My initial thought was to simply parallelize the
for loop in the random array generation function. This actually decrease performance by about 0.2 seconds.
I attempted to increase/decrease the block size, make the scheduling dynamic, but all attempts yielded a
similar performance. I could not understand why this was occurring, but my guess was the distribution method
in C++ either didn't support parallel functionality or the compiler could not figure out a way to parallelize the code.

I designed this function to replace the serial version. It uses the simple `std::srand` function and performs
much faster.

```cpp
template <typename T>
std::vector<T> generate_random_array_parallel(const T lo, const T hi, const std::size_t size) {
    auto_cpu_timer t;

    std::srand(std::time(nullptr));
    std::vector<T> numbers(size);

    #pragma omp parallel for
    for (size_t i = 0; i < numbers.size(); ++i) {
        numbers[i] = lo + std::rand() % hi;
    }

    return numbers;
}
```

Generating an array of random elements now completes in under a second. Do note that the line
`std::srand(std::time(nullptr))` should exist in main. Also, although I am using this for
my needs to showcase the prefix sum, I would prefer to find a way to speed up or improve the
first implementation of this function. I am not convinced that this implementation is thread-safe,
and believe to make things more random, especially across threads, it may be necessary to
have an engine per thread and not share them.

```bash
❯ /Users/kylehurd/Workplace/tmp/build/tmp
 0.728291s wall, 1.780000s user + 0.090000s system = 1.870000s CPU (256.8%)
```

### Final Implementation

In my final attempt at generating a more proper parallel version, I came up with the below implementation.
Since there isn't really a way to guarantee the engine is thread safe, I opted to
create an engine for each thread. It is almost twice as slow as the previous idea, but I believe
this one is actually thread-safe, and is more C++-like. Now onto the actual prefix sum!

```cpp
template <typename T>
std::vector<T> generate_random_array_parallel(const T lo, const T hi, const std::size_t size) {
    auto_cpu_timer t;

    static std::random_device random_device;
    static std::vector<std::mt19937> engines(omp_get_max_threads(), std::mt19937(random_device()));
    std::vector<T> numbers(size);

    #pragma omp parallel
    {
        const int thread_num{ omp_get_thread_num() };
        std::uniform_int_distribution<T> distribution(lo, hi);

        #pragma omp for
        for (std::size_t i = 0; i < numbers.size(); ++i)
        {
            numbers[i] = distribution(engines[thread_num]);
        }
    }

    return numbers;
}
```


## Prefix Sum - Parallel

The parallel version of the prefix sum consists of two passes called an `upsweep` and a `downsweep`.

### Up Sweep

In the up sweep portion, it is needed to build a binary tree such that:

- The root is a sum between [x,y)
- If a node has a sum [lo,hi) and hi > lo
  - Left child sum is [lo, middle)
  - Right child sum is [middle, hi)
  - A leaf has a sum of [i, i + 1)

This approach is known as `divide and conquer`. It can be constructed in a parallel fashion.

```cpp
template <typename T>
static void upsweep_parallel(const std::vector<T>& input, std::vector<std::vector<T>>& tree) {

    #pragma omp parallel for
    for (std::size_t i = 0; i < input.size(); ++i) {
        tree[0][i] = input[i];
    }

    for (int h = 1; h < tree.size(); ++h) {
        const int i_end(n / std::pow(2, h));

        #pragma omp parallel for
        for (int i = 0; i < i_end; ++i) {
            tree[h][i] = tree[h - 1][i << 1] + tree[h - 1][(i << 1) + 1];
        }
    }
    return;
}
```

### Down Sweep

The down sweep is another `divide and conquer` approach and is relatively straightforward to
parallelize.

- Root is given a left value of 0
- Given a left value
  - Pass the left child the left value of the current node
  - Pass the right child the left value of the current node __plus__ the left child's sum
- For a given position `i`
  - output[i] = left + input[i]

```cpp
template <typename T>
static void downsweep_parallel(std::vector<T>& input, std::vector<std::vector<T>>& tree1, std::vector<std::vector<T>>& tree2) {
    tree2[tree2.size() - 1][0] = 0;
    for (int h = tree1.size() - 2; h >= 0; --h) {
        const int i_end(std::ceil(input.size() / std::pow(2, h)));

        #pragma omp parallel for
        for (int i = 0; i < i_end; ++i) {
            tree2[h][i] = i % 2 == 0 ?
                tree2[h + 1][i >> 1] :
                    tree2[h + 1][(i - 1) >> 1] + tree1[h][i - 1];
        }
    }

    for (std::size_t i = 0; i < input.size(); ++i) {
        input[i] += tree2[0][i];
    }
    return;
}
```

### Combine Up Sweep and Down Sweep

The result is this:

```cpp
template <typename T>
std::vector<T> prefix_sum_parallel(const std::vector<T>& input) {
    auto_cpu_timer t;

    if (input.size() == 0) {
        return {};
    }

    const std::size_t size_of_trees(1 + std::ceil(std::log(input.size() / std::log(2))));

    std::vector<T> result(input.size());
    std::vector<std::vector<T>> tree1(size_of_trees, std::vector<T>(input.size()));
    std::vector<std::vector<T>> tree2(size_of_trees, std::vector<T>(input.size()));

    upsweep_parallel(input, tree1);
    downsweep_parallel(result, tree1, tree2);

    return result;
}
```
## Conclusion

The primary limitation to this design is memory. The larger the array, the more memory
is required. Which almost defeats the purpose of parallelizing the algorithm. The reality is,
at a certain point, memory and overhead gets in the way, making the parallel version no
faster than the serial version. But it is a fun exercise to bang your head on for days.
I sure had fun!

In my eyes, I do not see much benefit from the parallel implementation of the prefix sum.
The serial version for 100,000,000 numbers completes in a second. If there is a use for
numbers in the billions, it would most likely be a very specialized use-case. During this
coding challenge, I definitely learned more about how to properly generate random numbers!
