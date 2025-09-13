---
title: Looping Performance in C++
date: 2017-04-03 20:21:00
toc: true
comments: true
draft: true
categories:
- C++
tags:
- C++
---

Today I was testing the performance of a piece of code, which is basically accessing each element in a container within a for loop. But the result is quite shocking because I found the std::for_each version is 10 times faster than the raw loop. What?

<!-- more -->

## Test for yourself

```cpp
#include <vector>
#include <array>
#include <algorithm>
#include <chrono>
#include <iostream>

int main()
{
	std::array<std::vector<float>, 6> buffers;
	for (int i = 0; i < 6; ++i) {
		buffers[i] = std::vector<float>(480000, 0.5f);
	}

	auto tstart = std::chrono::high_resolution_clock::now();
	auto accum = 0;
	for (int i = 0; i < 6; ++i) {
		for (size_t j = 0; j < buffers[i].size(); ++j) {
			if (buffers[i][j] < 1.0f)
				++accum;
		}
	}
	auto tend = std::chrono::high_resolution_clock::now();
	auto duration = tend - tstart;
	std::cout << "Raw loop: " << std::chrono::duration_cast<std::chrono::milliseconds>(duration).count() << std::endl;

	tstart = std::chrono::high_resolution_clock::now();
	accum = 0;
	for (const auto& buffer : buffers) {
		for (const auto& value : buffer) {
			if (value < 1.0f)
				++accum;
		}
	}
	tend = std::chrono::high_resolution_clock::now();
	duration = tend - tstart;
	std::cout <<"Range-based for loop: " << std::chrono::duration_cast<std::chrono::milliseconds>(duration).count() << std::endl;

	tstart = std::chrono::high_resolution_clock::now();
	accum = 0;
	std::for_each(buffers.begin(), buffers.end(),
		[&accum](const std::vector<float>& buffer) {
			std::for_each(buffer.begin(), buffer.end(), [&accum](float value) { if (value < 1.0f) ++accum; });
		}
	);
	tend = std::chrono::high_resolution_clock::now();
	duration = tend - tstart;
	std::cout << "std::for_each: " << std::chrono::duration_cast<std::chrono::milliseconds>(duration).count() << std::endl;
}
```

I was using VS2015 under Debug build. Here's the output:

> Raw loop: 978

> Range-based for loop: 426

> std::for_each: 66

However when I switched to Release build:

> Raw loop: 2

> Range-based for loop: 2

> std::for_each: 5

That's what I've been expecting. And when I changed time precision to nanoseconds it turned out that raw loop is slightly faster than the range-based for loop.

## Lesson learned

The compiler sure knows how to optimize your code. So do your profiling with optimization on.
