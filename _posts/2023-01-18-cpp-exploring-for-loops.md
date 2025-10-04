---
title: Exploring For Loops in Cpp
last_modified_at: 2023-01-18T12:00:00-05:00
---

This week I was diving into some of the new features C++ has regarding iteration.
This newfound interest stemmed from wanting to make cleaner, more understandable
code. There have been many situations when doing LeetCode problems or personal
projects where I felt like I could clean up or use more modern tools in certain
situations. In this post, I will explore some new ways for iterating for loops. I
am using C++20.

## Basic Iteration

We all are familiar with the generic for loop like this:

```cpp
#include <iostream>

for (int i{ 0 }; i < 5; ++i) {
  std::cout << i << ',';
}
```
```bash
❯ /Users/kylehurd/Workplace/tmp/build/for-loops
0,1,2,3,4,
```

The above for loop will display to stdout the values from 0-4. For me, I do see
two issues that have always bugged me (like my joke?). The first is, for each iteration, we may want
to ensure that the value of `i` does not update within the for loop itself. With this
implementation there is not a good way to do this. The second is it doesn't
look clean to me. Surely there can be a better way to use an iterator.

As I did some digging, I found a new function in the C++20 standard called
[std::views::iota](https://en.cppreference.com/w/cpp/ranges/iota_view).
We can use that function to help try and clean up the for loop. In a basic setting,
it works very similarly to a [range for loop](https://docs.python.org/3/library/functions.html#func-range)
in Python. The first argument is the starting position and the second argument is the ending
position. The start is inclusive where as the end is exclusive. `[start, end)`

```cpp
for (const int i : std::views::iota(0, 5)) {
  std::cout << i << ',';
}
```

I personally find this easier to read, although the implementation is actually a bit lengthier
compared to the generic for loop. Plus, our iterator `i` is now a `const` type. There are other
reasons why this type of for loop would be ideal over the generic version. For example, if you
wanted to filter the iterations by odd numbers only you can add a filter option:

```cpp
auto odd = [](const int value) { return value % 2 == 1; };

for (const int i : std::views::iota(0,10)
                 | std::views::filter(odd)) {
  std::cout << i << ',';
}
```
```bash
❯ /Users/kylehurd/Workplace/tmp/build/for-loops
1,3,5,7,9,
```

There are much more modifications you can make within the `std::views` namespace. You can check
them out in the [Microsoft Documentation](https://learn.microsoft.com/en-us/cpp/standard-library/view-classes?view=msvc-170).

## Container Iteration

For this section, I focused on different ways you can iterate over containers. We can filter,
transform, and perform other operations on the container in the same way as above. For example,
given a `std::vector` object, we can do the following:

```cpp

#include <vector>

// ...

std::vector<int> v { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
const auto odd = [](const int number) { return number % 2 == 1; };

for (const int value : v | std::views::filter(odd)) {
  std::cout << value << ',';
}
std::cout << std::endl;
```
```bash
❯ /Users/kylehurd/Workplace/tmp/build/for-loops
1,3,5,7,9,
```

Shifting over to a different type, consider the following situation. Below we have an unordered
map containing a string as the key and float as the value. The key is the store that was
visited and the value is the amount spent at said store.

```cpp
std::unordered_map<std::string, float> transactions;
transactions.insert_or_assign("Safeway", 1.97);
transactions.insert_or_assign("Walmart", 18.81);
transactions.insert_or_assign("Apple", 119.03);
transactions.insert_or_assign("Sears", 32.43);
```

Given this map, if we wanted to iterate over each item and print each item, we could do this:

```cpp
for (const auto& transaction : transactions) {
  std::cout << transaction.first << ": $" << transaction.second << '\n';
}

// or we could utilize a lambda

const auto print_transaction = [](const auto& node) {
  std::cout << node.first << ": $" << node.second << '\n';
};

for (const auto& transaction : transactions) {
  print_transaction(transaction)
}
```

The lambda helps the for loop become slightly more readable, but I do not like the variables `.first`
and `.second`. To me, they make it unclear what first and second are. There is another way we can
iterate over a pair object like this by unpacking the map before entering the for loop. This is
one of my favorite ways to iterate over these types of objects at the moment. It allows us to
give better names to our variables before entering the loop.

```cpp
#include <format>

// ...

for (const auto& [store, cost] : transactions) {
  // gcc-13
  std::cout << std::format("{}: ${}\n", store, cost);
  // gcc-12
  std::cout << store << ": $" << cost << '\n';
}
```

That's my rundown on a few different ways to iterate! There are a lot of cool features released in C++20
and a lot of it helps improve readability of a program.
