---
title: Threading in C++
categories: [ programming ]
tags: [ cpp ]
last_modified_at: 2023-05-24-T12:00:00-05:00
---

This post stems from my recent exposure to handling asynchronous functions in other
programming languages. One of my recent projects I have dove into is
an Alexa Skill called [SkyBro](https://github.com/khurd21/SkyBro). In this project,
I used dependencies such as DynamoDB and other APIs that ate up precious time to delivering
a meaningful message back to a user. In order to speed up the Alexa Skill, writing asynchronous
code to gather the needed information became necessary.

I rather enjoyed writing asynchronous code. Also, knowing how strange C++ can be, I
decided it would be worthwhile to see how asynchronous code could look in the language.
I discovered it is fairly simple to set up. Here is the scenario I built:

## C++ Async Example

In this example, I used the following libraries and using statements:

```cpp
#include <iostream>
#include <chrono>
#include <thread>
#include <future>

#include <boost/timer/timer.hpp>

using boost::timer::auto_cpu_timer;
```

Let's say we have a function called `foo` and another called `bar`.

```cpp
int foo() {
    // Perform some time-consuming task
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return 42;
}

std::string bar() {
    // Perform some time-consuming task
    std::this_thread::sleep_for(std::chrono::seconds(3));
    return "Hello, world!";
}
```

Looking at the implementation of the two functions, although very trivial, we can see that
they are both unrelated to each other and also can take 2-3 seconds to return there result.
These two functions are great candidates to be run asynchronously! When I try to see if a function
is worth being run asynchronously, I typically look at the following criteria as a start:

- Is the function computationally heavy?
- Are there any other blockers that would prohibit the function from
  completing its computation?
- Are there multiple items that have to be computed before the program can continue?

In the `foo` and `bar` functions above, I am assuming both are computationally heavy. They take
several seconds to complete, and because there are more than one, this length becomes
more noticeable. There also aren't any blockers or dependencies the functions depend on.

## Serial Example

If we run these two functions serially, it'll take 5 seconds to complete.

### Code

```cpp
void example_sync() {
    auto_cpu_timer t;

    // Get function 1 and 2 results when called
    const auto result_foo{ foo() };
    const auto result_bar{ bar() };

    // Do something
    std::cout << "Performing other tasks....\n";

    // Use the results
    std::cout << "Function 1 result: " << result_foo << std::endl;
    std::cout << "Function 2 result: " << result_bar << std::endl;

    std::cout << "Exiting example sync....\n";
}
```

### Result

```shell
Starting example sync....
Performing other tasks....
Function 1 result: 42
Function 2 result: Hello, world!
Exiting example sync....
 5.010203s wall, 0.000000s user + 0.000000s system = 0.000000s CPU (n/a%)
```

## Asynchronous Example

The async example is fairly similar, with the added section of starting the tasks and then
waiting for their results when needed. Note the time to complete is 3 seconds which is the
longest computation coming from the `bar` function. This highlights that running code
asynchronously is only as fast as the slowest function. If there are many functions being run
at the same time, you could come across other constraints that can slow down the function
completions.

### Code

```cpp
void example_async() {
    auto_cpu_timer t;

    std::cout << "Starting example async....\n";

    // Start function 1 and 2 asynchronously
    std::future<int> result1{ std::async(std::launch::async, foo) };
    std::future<std::string> result2{ std::async(std::launch::async, bar) };

    // Do some other work while functions are running...
    std::cout << "Performing tasks while functions are running....\n";

    // Retrieve the results
    const auto result_foo{ result1.get() };
    const auto result_bar{ result2.get() };

    // Use the results
    std::cout << "Function 1 result: " << result_foo << std::endl;
    std::cout << "Function 2 result: " << result_bar << std::endl;

    std::cout << "Exiting example async....\n";
}
```

## Result

```shell
Starting example async....
Performing tasks while functions are running....
Function 1 result: 42
Function 2 result: Hello, world!
Exiting example async....
 3.005110s wall, 0.000000s user + 0.000000s system = 0.000000s CPU (n/a%)
```

## Future Discussions

Although this is a great functionality, and the async operations are relatively simple to read,
for me coming from C#, this structure seems like it could be fairly simple to accidentally
create a blocking operation. I think I much prefer the use of declaring `async` in the function
header the way Python or C# do. Although, it does look like C++20 has a similar feature that
gets closer to this called [coroutines](https://en.cppreference.com/w/cpp/language/coroutines).
This does not look as simple as how other languages handle things. I found another site that
discusses coroutines in much depth [here](https://lewissbaker.github.io).
