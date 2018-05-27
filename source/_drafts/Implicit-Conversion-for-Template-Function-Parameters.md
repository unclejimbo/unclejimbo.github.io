---
title: Implicit Conversion for Template Function Parameters
date: 2018-03-07 16:13:34
categories:
- C++
tags:
- C++
---

I encountered this problem today when I tried to write a generic file reader. At the bare minimum it looks like,

```cpp
template<typename T>
void read_some_file(const std::string& file_name,
                    std::vector<T>& essential,
                    boost::optional<std::vector<T>&> optional)
{
    // reading...
}
```

This interface is pretty clear and says everything about this function. It reads from a file and store some values into essential, while some other values are optional which the user can choose to store or discard. I use a template function here to allow it to work with different data types.

However, when I tried to use this function,

```cpp
std::string file_name;
std::vector<float> essential;
std::vector<float> optional;

read_some_file(file_name, essential, optional); // bang!
```

the compiler generated a template argument deduction error:
> template argument deduction/substitution failed
>
> 'std::vector<float> is not derived from 'boost::optional<std::vector<T>&>'

But why? I thought the compiler was clever enough to deduce the template parameter T in a simple case like this.

<!---more--->

# Template Argument Deduction Revisited

