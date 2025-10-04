---
title: Adding Boost and OpenMP Library to CMake Project
last_modified_at: 2023-02-09-T12:00:00-05:00
---

Starting around mid-2021, I discovered a website called <a href="https://projecteuler.net">Project Euler</a>.
Project Euler "is a series of challenging mathematical/computer programming problems that will require more
than just mathematical insights to solve." The questions tend to take me 30 minutes to several hours to come up with a reasonable
solution to get the answer. The website encourages that users implement solutions that run in under a minute.
This drove me to consider tools that help speed up the runtime of the program like
<a href="https://www.openmp.org/resources/refguides/" target="_blank">OpenMP</a>. Other questions generated
results that exceeded the storage of 64 bit variables, which made me consider other tools such as
<a href="https://www.boost.org/doc/libs/1_81_0/libs/multiprecision/doc/html/boost_multiprecision/tut/ints/cpp_int.html"
    target="_blank">
cpp_int</a> to solve them.

In my first attempt at using these tools, I did not utilize a build manager like cmake or boost-build. I relied on
compiling with just the gcc compiler, and had to add links to the libraries I wanted to include. This was fine for
solving Project Euler questions, however as I solved more problems I struggled to remember what was needed to properly
compile each question. Additionally, from transitioning from a Mac with an Intel processor to the new ARM processor, the
original set up I had no longer functioned. It was becoming tedious to program, so I elected to figure out how to use
cmake to include the Boost library and OpenMP into my project.


## Compiler Choice

Wow. Compilers are finicky. I have always used `gcc` to compile my code. However, when trying to make a workable
environment, I discovered `gcc` for the tools I was introducing was not the best choice for developing. It would be
possible, but it would require a lot more steps that I was unwilling to take. I will describe the problems I
faced below, but for now I decided to use the Homebrew version of Clang++.

I used homebrew to download the Boost library. However, this library was compiled using Clang. It was possible to
recompile using the gcc compiler instead, but this ended up breaking other tools that relied on Boost. In order to
continue to use GCC, I would have needed to have to compilations of the library: one using Clang, the other using GCC.
I did not want to manage this, so I opted to switch to Clang.

The final issue, Apple's Clang does not support OpenMP. This led me to finally choosing the Homebrew's Clang. This version
is compatible with the brew installation of Boost and also with OpenMP.

```bash
╰─❯ brew install llvm
```

This was the only installation I needed to get this compiler. As for using this compiler in cmake, I added these two
setter commands to the `CMakeLists.txt`.

```cmake
set(CMAKE_CXX_COMPILER <PATH_TO_CXX_COMPILER>)
set(CMAKE_C_COMPILER <PATH_TO_C_COMPILER>)
```

To find the path to the clang compiler, I ran the following commands:

```bash
❯ brew info llvm
==> llvm: stable 15.0.7 (bottled), HEAD [keg-only]
Next-gen compiler infrastructure
https://llvm.org/
/opt/homebrew/Cellar/llvm/15.0.7_1 (6,411 files, 1.3GB)
  Poured from bottle on 2023-02-04 at 20:59:32
From: https://github.com/Homebrew/homebrew-core/blob/HEAD/Formula/llvm.rb
License: Apache-2.0 with LLVM-exception
==> Dependencies
Build: cmake ✔, swig ✘
Required: python@3.11 ✔, six ✔, z3 ✔, zstd ✔
### ... Redacted ... ###

```
In my case, the two compilers existed in these directories:

```text
/opt/homebrew/opt/llvm/bin/clang++
/opt/homebrew/opt/llvm/bin/clang
```

## Boost

Boost was relatively simple to install on MacOS. Here is the command:

```bash
╰─❯ brew install boost
```

```bash
❯ brew info boost
==> boost: stable 1.81.0 (bottled), HEAD
Collection of portable C++ source libraries
https://www.boost.org/
/opt/homebrew/Cellar/boost/1.81.0_1 (15,831 files, 481.6MB) *
  Poured from bottle on 2023-02-04 at 21:01:32
From: https://github.com/Homebrew/homebrew-core/blob/HEAD/Formula/boost.rb
```

When using the Homebrew version of the Clang compiler, this should be a simple onboarding process to a project.
It is important when running the `find_package` command that you include the components required for the
specific project. In my case, I have only been relying on `Boost::timer`. More information can be found
<a href="https://cmake.org/cmake/help/latest/module/FindBoost.html" target="_blank">here</a>.

For Project Euler, these two lines satisfied my needs.

```cmake
find_package(Boost 1.81 REQUIRED COMPONENTS timer)
target_link_libraries(${PROJECT_NAME} PRIVATE Boost::timer)
```

## OpenMP

For OpenMP, it may or may not be required to have this installation:

```bash
brew install libomp
```

This was a strange problem to solve. When I ran the program without searching for the OpenMP library,
Clang gave zero complaints about running, even though the library could not be found. However, CMake delivers
a nice message indicating which libraries were found and which ones were not.

For adding OpenMP to my project, it turns out all I need after the Homebrew install is to add the following lines
in the `CMakeLists.txt` file.

```cmake
find_package(OpenMP)
if (OpenMP_CXX_FOUND)
    target_link_libraries(${PROJECT_NAME} PUBLIC OpenMP::OpenMP_CXX)
endif()
```

## CMake File

The end result of this led to a CMake file that looks like the one below.
It is relatively simple, and was just added to speed up my development and to standardize how I run each
project euler question. Using CMake also helped my ability to diagnose problems a lot faster than
relying on the error or warning messages from the compiler. In addition to Clang giving very meaningful
alerts in the terminal window, CMake is able to integrate with my favorite code editor, VSCode, giving
helpful information after building the project.

For the `add_executable`, I simply change the file name for the specific question I am attempting to answer
or simply wish to review.

```cmake
cmake_minimum_required(VERSION 3.25)

set(CMAKE_CXX_STANDARD 20)
set(Boost_USE_STATIC_LIBS OFF)
set(Boost_USE_MULTITHREADED ON)
set(Boost_USE_STATIC_RUNTIME OFF)

set(CMAKE_CXX_COMPILER /opt/homebrew/opt/llvm/bin/clang++)
set(CMAKE_C_COMPILER /opt/homebrew/opt/llvm/bin/clang)

project(project-euler)

add_executable(${PROJECT_NAME} 31-40/31.cpp)

find_package(Boost 1.81 REQUIRED COMPONENTS timer)
find_package(OpenMP)
if (OpenMP_CXX_FOUND)
    target_link_libraries(${PROJECT_NAME} PUBLIC OpenMP::OpenMP_CXX)
endif()
target_link_libraries(${PROJECT_NAME} PRIVATE Boost::timer)
```

## Conclusion

In conclusion, I should have dove into CMake a long time ago. Although I feel like interacting with code
at a terminal level is really important and helped me considerably in learning new languages such as C/C++, build
tools like CMake really help the programmer fret less about nitty-gritty technical details that don't necessarily
target what it is you are trying to accomplish. At the end of the day, I was trying to get better at algorithms
and mathematics and the tools I was using was limiting how quickly I could I could get better at those. Plus, you are
still able to communicate with CMake at the terminal shell level! I also think tools such as Boost and OpenMP are
important to have in a programmers skill set, and using CMake to more easily incorporate them into a project allows
developers to use them with less headache and setup!
