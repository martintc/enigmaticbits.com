---
layout: ../../layouts/post.astro
title: 'Intro to Pointers 2'
author: 'Todd'
pubDate: 2022-09-27
---

#  Pointers: Math with pointers

In a previous post, I covered how variables and memory related by using pointers. It was covered that variables are a generic and short hand way of representing memory locations inside of a computer. When a program is executed, the computer will delegate memory to a program and each piece of data of a program is store in at it’s own address.

We also covered that arrays were pretty much just a collection of memory addresses placed conveniently next to one another. When iterating through an array, the computer is taking the address of the first index and added the bandwidth of the data type in order to achieve another index.

Not only can the computer do this math with memory addresses, a programmer can directly do this also. Take a look at the following source code and it’s output:

```c
  #include <stdio.h>

  int main () {

        int a[] = {1, 2, 3, 4};

        int *p = a;

        printf("This is the address of a[0]: %p\n", p);
        printf("At address: %p is %d\n", p, *p);
        printf("At Address: %p is %d\n", (p+4), *(p+4)); // error
        printf("At Address: %p is %d\n", (p+1), (*p+1));

        return 0;

  }

  ```

```
  This is the address of a[0]: 0x7ffca894b0c0
  At address: 0x7ffca894b0c0 is 1
  At Address: 0x7ffca894b0d0 is -1466650176
  At Address: 0x7ffca894b0c4 is 2
```

In the code, a pointer variable is defined. A pointer variable is a variable of a type that points to an address of another variable. The type of a pointer variable is important, this tells the computer that we know that the memory location we are pointing at is 4 bytes in width, since an integer type (at least on my machine) is 4 bytes wide.

In the third line of code, I added “1” to the address stored in the pointer variable. Since the pointer is of type integer, it knows to add 1 in respect to the width of an integer variable. So in reality, 4 is added.

With this pointer variable, we are able to use simple math operations to move around. In the second line of output, I added 4 to the address stored which went out of bounds. Remember, in C, the compiler will not protect from these kinds of mistakes. Without careful controls put in by the programmer a C program can iterate outside of the bounds of an array. In this case, when I added 4, I actually added 4 * 4 which is 16 bytes. If the array had 5 indexes, we would have gotten a value that would have made sense.


