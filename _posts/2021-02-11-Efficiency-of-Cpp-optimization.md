---
layout: post
title: "Efficiency of C++ code optimization"
author: "Dominik Adamski"
---

C++ code can be highly optimized. This blog post shows how compilers deal with optimization of the source code. It presents simple code snippets and analyzes how different compilers can optimize it. x86-64 platform was chosen as the target platform. Assembly was generated on [Compiler Explorer - godbolt.org](https://godbolt.org).

## As-if rule

Efficiency of the program execution has been always one of the major aspects for C++ commitee. That's why the  C++ standard states, that compilers are allowed to perform any optimization as long as the observable output is the same as if no code optimization is performed. Compilers have to meet the following conditions:

1. read/write accesses to volatile objects are not reordered and they are done in the same way as they are implemented in the code
2. at the end of the program all data is written  to output files in the same order as if unoptimzed code is executed
3. user messages prompts are displayed earlier than corresponded input form

There are two exceptions from the rules mentioned above. Compilers can skip copy/move operations and they can reduce number of new expressions (since C++14) even if it can lead to change of the observable behavior.


## Removal of unused assignment

Code from the listing below contains unnecessary loop. Forthe function `foo` returns the final value of the loop counter. Assignment in the loop body is unnecessary because it has no effect on observable behavior of the function `foo`.

```cpp
int foo() {
  int a[10];
  int i;
  for (i = 0; i < 10; ++i)
    a[i] = i;
  return i;
}
```

If no optimization is enabled GCC 10.2 then [generated assembly output reflects input code](https://godbolt.org/z/6P7qcx). It is enough to turn on at least -O1 flag  and [GCC 10.2 removes all unnecessary code](https://godbolt.org/z/GKjTWo):

```assembly
foo():
        mov     eax, 10
        ret
```

## Optimization of STL arrays

C++11 introduced array container and lambda functions. Array container allows developer to eliminate error prone raw index manipulation. Function `foo_array` implements the same functionality as previous one but it uses more sophisticated language expressions (template code, iterators, lambda function).

```cpp
#include <array>
#include <algorithm>

int foo_array() {
  std::array<int, 10> a;
  int i = 0;
  std::for_each(a.begin(), a.end(), [&i](int &t) { t = i++; });
  return i;
}
```

Although input code was syntactically complex, [GCC 10.2 can remove almost all unnecessary code for `-O1` optimization](https://godbolt.org/z/rG45x7):

```assembly
foo_array():
        mov     eax, 10
.L2:
        sub     rax, 1
        jne     .L2
        mov     eax, 10
        ret
```

[Clang 11.0.1 cannot eliminate all unnecessary function calls](https://godbolt.org/z/5e4qd8) (i.e. begin/end iterator and lambda call) for `-O1` optimization. For `-O2` optimization flag both compilers produce the assembly as for optimized C-style `foo` function.

## Optimization of new expression

C++14 standard allows compiler to eliminate redundant `new` and `delete`  expressions even if such operation breaks `as-if` optimization rule. In comparison to partially mandatory copy elision, allocation optimization is optional and not all compilers implement this feature (MSVC does not support it). Compilers which support this feature (i.e. GCC, Clang) provide different levels of optimization. Functions `foo_new_expr` and `foo_make_unique` illustrate this issue.

```cpp
#include <memory>

int foo_new_expr() {
  int i = 0;
  auto a = new int[10];
  delete[] a;
  return i;
}

int foo_make_unique() {
  int i;
  auto a = std::make_unique<int[]>(10);
  for (i = 0; i < 10; ++i) {
    a[i] = i;
  }
  return i;
}
```
`make_unique` uses internally `new` expression. That's why it can be optimized similarly as raw `new` expressions. [GCC 10.2 can properly eliminate new expression only for `foo_new_expr` function](https://godbolt.org/z/vh8vao), whereas Clang 11.0.1 can simplify not only trivial `foo_new_expr` function but also more sophisticated `foo_make_unique`. [When `-O3` flag is enabled then Clang can produce the same assembly as for `foo` function.](https://godbolt.org/z/M9W46h).

It should be noted that compiler is only allowed to modify `new` and `delete`  expressions (i.e. `auto a = new int[10];`). If developer calls operator function then compiler is not allowed to elide allocation. Compiler can still perform some optimizations if the implementations of `operator new` and `operator delete` are known during compilation time. In such situation compiler can analyze function call and it can determine if further optimization is possible because it does not change an observable behaviour.

## Loop unroll

Previous code samples only write data to the array `a`. Compilers can easily detect that there are no readouts of array `a` items. In such situation any write operation to the array `a` can be skipped because it has no efect on observable behavior.

Compiler cannot simply remove `for` loop in the example below, because now the return value of the function `foo_loop` depends on the last element of the array `a`.

```cpp
#define LOOP_SIZE 10
int foo_loop() {
  int i = 0;
  int a[LOOP_SIZE];
  for (i = 0; i < LOOP_SIZE; ++i)
    a[i] = i;
  return a[LOOP_SIZE - 1];
}
```

It occurs that [GCC 10.2 can remove loop if `LOOP_SIZE` is small](https://godbolt.org/z/76WYzb) (when it is smaller than 72). `for` loop is executed [if `LOOP_SIZE` is greater than 72](https://godbolt.org/z/5qf8fr).

Although data access pattern is the same for both cases, compiler generates different code. It is caused by different optimization strategies. GCC cannot simply analyze loop code as humans do. If loop is simple and it contains small number of iterations then it can be unrolled. Loop unrolling operation transforms loop into set of consecutive intructions. Listing of function `foo_loop_unrolled` describes this optimization step.

```cpp
#define LOOP_SIZE 10
int foo_loop() {
  int i = 0;
  int a[LOOP_SIZE];
  // Replace loop by consecutive instructions
  a[0] = 0;
  a[1] = 1;
  a[2] = 2;
  a[3] = 3;
  a[4] = 4;
  a[5] = 5;
  a[6] = 6;
  a[7] = 7;
  a[8] = 8;
  a[9] = 9;
  // End of loop
  return a[9];
}
```
After unrolling compiler clearly sees that only `a[9]` element is further used in `return` statement. In consequence it can remove other assignments. Loop unrolling is powerful technique which allows not only to simplify loops but also to increase SIMD instruction usage (this process is called vectorization).

Unfortunately wrong loop unrolling can lead to increased size of output binary. That's why compilers have some predefined thresholds which heurestically define which loop unrolling can be beneficial. If loop is too big then it is not unrolled even if it can be easily simplified as it was shown in the examplary `foo_loop` function.

## Final remarks

Compilers can effectively optimize source code. Presented code samples show that developer cannot assume a priori what will be the outcome of the optimization. This outcome can vary significantly for different compilers. In consequence code benchmarking is the only way to measure the quality of generated binaries.

## Bibliography
* [https://en.cppreference.com/w/cpp/language/new](https://en.cppreference.com/w/cpp/language/new)
* [https://en.cppreference.com/w/cpp/language/copy_elision](https://en.cppreference.com/w/cpp/language/copy_elision)
* [https://en.cppreference.com/w/cpp/memory/unique_ptr/make_unique](https://en.cppreference.com/w/cpp/memory/unique_ptr/make_unique)
* [https://en.cppreference.com/w/cpp/language/as_if](ttps://en.cppreference.com/w/cpp/language/as_if)
* [https://llvm.org/docs/Passes.html#loop-unroll-unroll-loops](https://llvm.org/docs/Passes.html#loop-unroll-unroll-loops)
