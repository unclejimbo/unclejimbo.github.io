---
title: Eigen Cache Performance
date: 2018-06-28 23:06:30
categories:
- C++
tags:
- C++
- Eigen
- Cache
---

The storage order of Eigen matrix will hit your cache performance, depending on how you access the elements as well as the size of your matrix, although in a bit unintuitive way.

<---! more--->

# Row-major v.s. Column-major

It's recommended in Eigen that you should store the data of a dense matrix in the same order as you process them. For example, the codes bellow try to access the elements row by row so we would assume that the row-major version would run faster, because of cache coherency.

```cpp
#include <Eigen>
using namepsace Eigen;
using RowMat = Matrix<float, Dynamic, Dynamic, RowMajor>;
using ColMat = Matrix<float, Dynamic, Dynamic, ColMajor>;
const int ncol = 400;
const int nrow = 400;

int main()
{
    RowMat rowmat;
    ColMat colmat;
    rowmat.resize(ncol, nrow);
    colmat.resize(ncol, nrow);

    for (auto i = 0; i < nrow; ++i) {
        for (auto j = 0; j < ncol; ++j) {
            rowmat(i, j) = 1.0f;
        }
    }

    for (auto i = 0; i < nrow; ++i) {
        for (auto j = 0; j < ncol; ++j) {
            colmat(i, j) = 1.0f;
        }
    }

    return 0;
}
```

However, if you use `<chrono>` to profile the running time and it turns out that the column major version runs slightly faster than the other one. Why?

# Into the Cache

The image below shows how elements are stored in the cache, grids with the same color belong to the same row in a matrix. The top one shows the column major matrix and the bottom one shows the row major matrix. It's quite clear that the row major matrix is much more cache friendly since it reads a chunk of elements into a cache line and process the elements one by one. On the other hand, the column major matrix reads a chunk of elements into a cache line but only process the first element which belongs to the first row and then it needs to fetch the other row. The cache misses are very high. Or are they?

![](http://7xllm5.com1.z0.glb.clouddn.com/cache.png)

Let's hit up a profiler, e.g. valgrind, and here's what I get when I use it to monitor the cache usage using the command

```
valgrind --tool=callgrind --simulate-cache=yes a.out
```

| Storage Order | Dw    | D1mw  | DLmw  |
| ------------- | ----- | ----- | ----- |
| Column-major  | 32000 | 10000 | 10000 |
| Row-major     | 32000 | 10400 | 10000 |

In the table above, **Dw** represents data write access, **D1mw** represents L1 data cache write misses and **DLmw** represents L3 data cache write misses. It's clear that the row-major version has 400 more cache misses. The reason behind this is actually system specific. I'll present my cpu information here using `lscpu`,

```
L1d cache: 32K
L1i cache: 32K
L2 cache:  256K
L3 cache:  8192K
```

Since the cache line size of most cpus is 64 Bytes, and the L1 data cache is 32 KB, so my cpu has 512 cache lines in the L1 data cache. In another word, all the columns could fit into the L1 cache without any trouble and we won't get any cache misses after they are read into the cache. So it's like we'll get 1 cache miss in every 16 cache writes, precisely. In the row major case however,
