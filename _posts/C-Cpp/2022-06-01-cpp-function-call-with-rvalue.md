---
title: Cpp Function Call With Rvalue
date: 2022.06.01
categories: [C-Cpp]
tags: [c-cpp]
---

## `shared_ptr`

I met a problem when I initialize an object.

```cpp
std::shared_ptr<Logger> logger(new Logger("root"));
```
```
error: no matching function for call to ‘mysylar::Logger::Logger(mysylar::Logger*)
```

It is caused by passing rvalue to the constructor. The following code can be compiled successfully.

```cpp
std::string name = "root";
std::shared_ptr<Logger> logger(new Logger(name));
```

Use c++11 standard also can solve this problem.