---
layout: post
title:  "Modern Error Handling in C++"
date:   2017-06-03 21:06
categories: c++ programming
---

This post provides a broad overview of error handling in C++ in general and how
the `expected<T, E>` proposed for standardisation will contribute to that big menu 
of error handling design patterns available to the C++ programmer.

## C Error handling - A.K.A Integer return codes
Historically C++ 98 code has taken one of two design patterns when returning errors 
from functions. This pattern is taken from C, and indeed is pure C:

```C++
struct handle
{
  int fd;
};
enum errors
{
  SUCCESS=0,
  NOMEM,
  NOTFOUND
};

extern int openfile(struct handle **outh, const char *path)
{
  *outh = malloc(sizeof(struct handle));
  if(!*outh)
    return NOMEM;
  (*outh)->fd = open(path, O_RDONLY);
  if((*outh)->fd == -1)
  {
    free(*outh);
    *outh = NULL;
    return NOTFOUND;
  }
  return SUCCESS;
}
```

Variations on the same pattern are returning the `enum` type directly, returning a 
boolean and using a thread locally stored global variable such as `errno` and so on.

## C++ 98 style error handling: throwing exceptions

The second C++ 98 design pattern ought to also be very familiar to readers. 
If an error occurs, throw an exception to indicate the problem. Throw for
everything.

```c++
// Abstract base class for some handle implementation
struct handle
{
  int fd;
  virtual ~handle()
  {
    if(fd != -1)
    {
      close(fd);
      fd = -1;
    }
  }
};
class handle_ref;  // Some sort of smart pointer managing a handle pointer

extern handle_ref openfile(const char *path)
{
  int fd = open(path, O_RDONLY);
  if(fd == -1)
  {
    throw std::runtime_error("File not found");
  }
  // RAII close the file if exception throw
  handle temp(fd);
  // Could throw std::bad_alloc or any other kind of exception during construction
  return handle_ref(new some_derived_handle_implementation(temp));
}
```
Old hands will quite correctly chafe at the use of `std::runtime_error` to report a file
not found, but nevertheless such a design pattern is very common in the wild, especially
in older C++ which went a bit mad on using exception throws for control flow.



## C++ 11 style error handling: error_code and noexcept


