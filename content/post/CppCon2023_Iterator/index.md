---
title: C++ 中的迭代器
description: 对 CppCon 2023, Nicolai Josuttis 演讲的翻译与总结。
slug: CppCon 2023 Iterator
date: 2024-02-20 00:00:00+0000
image: cppcon2023-cover.png
categories:
    - cppcon
    - techs
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

[Back to Basics: Iterators in C++ - Nicolai Josuttis - CppCon 2023 - YouTube](https://www.youtube.com/watch?v=26aW6aBVpk0)

[Silides](https://github.com/CppCon/CppCon2023/blob/main/Presentations/B2B_Iterators.pdf)

CppCon 2023，一期 Back to basics，讲迭代器的。感觉应该比较有用

## 循环访问数组

当我们循环访问数组时，我们最初学到的是使用 index

```cpp
int arr[] = {10, 20, 30, 40};

for (int i = 0; i < 4; ++i) {
    std::cout << arr[i] << '\n';
}
```

当然也可以通过指针

```cpp
// 注意 int* p = arr+4 is ok
// 但此时 *p 是 ub
for (int* p = arr; p < arr + 4; ++p) {
    std::cout << *p <<'\n';
}
```

迭代器是通过指针抽象而来的，

那么我们如何通过指针访问一个数据结构呢？比如 vector，我们如何知道地址的起始和结束？

答：通过一些成员函数，询问 vector

```cpp
std::vector<int> v {1, 2, 3, 4, 5};
for (std::vector<int>::iterator pos = v.begin(); pos < v.end(); ++pos) {
    std::cout << *pos << '\n';
}
```

对于 string，也是同理。

当然，C++11 之后可以写 auto，就没人这么写了。

此外，这里还有一个规则***half-open range***，也就是说 end() 并不是一个有效的值。

为什么这么干呢？和前面指针的例子是同理，首先我们可以表示它，只是不能访问。此外我们没有表示空的办法，但有了这个规则就可以通过 begin() == end() 表示空。

## Why Iterator

### Index Operator vs. Iterator

首先考虑 vector，对于 vector 来说

```cpp
std::vector<int> v {1, 2, 3, 4, 5};
std::cout << vec[3];
```

通过 `[]` 访问很简单，首先是因为我们知道其中每个元素的大小，其次 vector 是内存连续的，很容易就能计算出位置。

那么再考虑一个不连续的容器，如 list

```cpp
std::list<int> lst{1, 2, 3, 4, 5};
std::cout << lst[3];
```

这种行为开销就大一点，因为我们并不能计算，只能挨个迭代，所以标准决定不提供这种操作。

也就是说，二者有语义上的不同，`operator []` 在语义上代表直接访问，迭代器有一种我们挨个访问的感觉。

### Iterator with Generic Code

所有的标准库容器迭代器 API 都提供：

- begin(), end() 来生成迭代器
  - 支持运算符 `++`, `!=`, `*`, ...

所以这更方便我们写泛型代码，不管是什么数据结构都可以使用统一的接口访问容器元素。如果在 C++20 中，就可以写：

```cpp
void printElems(const auto& coll) {
    for (const auto& elem : coll) {
        std::cout << elem << '\n';
    }
}
```

### auto 和 cbegin(), cend()

为什么会有 const 迭代器？他们不是为了 make ur life hard，是为了让你在编译器找到bug。

- 为了让 auto 支持只读迭代器，我们有：
  -  `const_iterator` -> 即，迭代器访问的对象是 const 的，代表我们不想更改元素，也可以检测我们是否意外更改了元素。
  - cbegin(), cend(), crbegin(), crend()

## 不同种类的迭代器

- **随机访问**（Random Access）迭代器

  可以直接访问另外一个位置。=,*, ++, ==, !=, --, +=, -=, <, <=, ...[], -

  vector, array, deque, raw arrays, strings

- **双向**（Bidirectional）迭代器

  可以前向也可以后向迭代

  =, *, ++, ==, !=, --

  list, 关联容器 (set map ...)

- **前向**（Forward）迭代器

  只能前向迭代

  =, *, ++, ==, !=

  forward_list, unordered_map

- 输入（Input）迭代器

  只能读一次元素

  istream_iterator<>

注意只有随机迭代器才能 +=

C++20 后，引入了 **连续**（Contiguous）range/迭代器

- 跟随机访问迭代器一样，同样可以直接访问其他位置

  可能是裸指针， range 提供 std::ranges::data()

  vector, array, raw arrays, strings（与随机迭代器不同的是，没有 deque）

## 迭代器和算法

算法提供了相同的迭代器 API，迭代器在这里作为容器和算法之间的粘合剂。

- 处理 half-open 范围内的元素
- 使用迭代器接口
- 泛型

## 迭代器的坑

### vector

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> coll{1, 2, 3, 5, 8, 9, 11, 13, 17};
    for (int elem : coll) {
        std::cout << elem << ' ';
    }
    
    auto pos8 = std::find(coll.begin(), coll.end(), 8);
    if (pos8 != coll.end()) {
        std::cout << "8 found\n";
        *pos8 *= 2;
    }
    
    coll.push_back(15); // 假设这里重新分配了内存，那么显然再对 pos8 访问就会 ub 
}
```

### transform with 4 args

std::transform 做的就是读取一个范围内的数据，然后做变换，最后写入。

```cpp
#include <iostream>
#include <list>
#include <vector>
#include <algorithm>

int square(int val) {
    return val * val;
}

void foo() {
    std::list<int> src;
    std::vector<int> dest;
    
    // 覆盖，而不是插入
    // 条件：dest.size() >= src.size()，否则是 ub
    std::transform(src.begin(), src.end(), dest.begin(), &square);
}
```

- 输出迭代器

  指向一个元素，且不知道范围在何处结束

  类似于原始指针

  **必须有元素**被覆盖（需要 resize， not reacllocate）

还有一种特殊的迭代器

- Inserters
  - 这种迭代器知道迭代的对象的信息
  - Insert 而不是覆盖

```cpp
#include <iostream>
#include <list>
#include <vector>
#include <algorithm>

int square(int val) {
    return val * val;
}

void foo() {
    std::list<int> src {1, 2, 3, 4, 5, 6};
    std::vector<int> dest;
   
    std::transform(src.begin(), src.end(), dest.begin(), &square);	// error
    
    std::vector<int> d2;
    std::transform(src.begin(), src.end(), std::back_inserter(d2), &square); //ok，会调用 push_back
}
```

### remove

```cpp
int main() {
	std::list<int> coll{6, 5, 4, 3, 2, 1, 1, 2, 3, 4, 5, 6};
    
    // remove all elements with value 3
    std::remove(coll.begin(), coll.end(), 3);
    
    // 会输出什么呢？
    // 6, 5, 4, 2, 1, 1, 2, 4, 5, 6, 5, 6
    // 3 确实没了，但是元素的个数是一样多的
}
```

- **Removing** 算法 **不会 remove**

  相反，它们会**替换应该被 removed 的值**，并且**返回一个新的 end**

  原因是迭代器是操作元素的，而不是操作容器的

  迭代器只能读、写以及移动

所以正确的做法是，获取 std::remove 返回的 new end，然后无视剩下的值。但这样非常 C++98/11

如果是 C++20，则可以

```cpp
int main() {
	std::list<int> coll{6, 5, 4, 3, 2, 1, 1, 2, 3, 4, 5, 6};
    
    // remove all elements with value 3
    auto new_end = std::remove(coll.begin(), coll.end(), 3);
    
    for (int elem : std::ranges::subrange(coll.begin(), new_end)) {
        std::cout << elem << ' ';
    }
}
```

如何理解range 呢？可以看之前的文章，总之 就是两个指针之间的 view，只是指向原来的数据，并不做 copy

当然，既然有了 20，那就有更好的方式了

```cpp
int main() {
	std::list<int> coll{6, 5, 4, 3, 2, 1, 1, 2, 3, 4, 5, 6};
    auto not3 = [](const auto& elem) {
      	return elem != 3;  
    };
    
    for (int elem : coll | std::views::filter(&not3)) {
        std::cout << elem << ' ';
    }
}
```

对于 range，有一个重要的特性，那就是其 cache begin()

```cpp
void print(const auto& elem) {
    for (const auto i : elem) {
        std::cout << i << ' ';
    }
    std::cout << '\n';
}

int main() {
	std::vector<int> coll {1, 4, 7, 10};
    print(coll);
    // 1 4 7 10
    
    auto is_even = [] (auto&& i) { return i % 2 == 0; }
    auto coll_even = coll | std::views::filter(&is_even);
    
    // 增加偶数
    for (int& i : coll_even) {
        i += 1;
    }
    print(coll);
    // 1 5 7 11
    
    for (int& i : coll_even) {
        i += 1;
    }
    print(coll);
    // 1 6 7 11 
    // 注意，后面的11此时不是偶数，但是 range 会 cache begin()，刚才的begin 是 这个位置，他还是增加了。
    // 如果修改后的值不符合谓词，那么就是 ub
}


```

## 总结

迭代器

- C++ 中关键的角色，是 ranges 和算法的粘合剂
- 纯粹的抽象，任何表现像是迭代器的就是迭代器
- 有不同的种类，其有不同的能力
- 总体来说，他们不知道表示的范围，不知道哪里是结束，不能 insert/remove
- 使用迭代器时，注意 range 要有效，不要把指向不同 range 的迭代器作比较