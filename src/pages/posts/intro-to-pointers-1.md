---
layout: ../../layouts/post.astro
title: 'Intro to Pointers 1'
author: 'Todd'
pubDate: 2022-09-27
---

# Pointers: The relationship between memory and variables
One of the most difficult subjects new programmers may be surprising to some. Some might think it is good software design practices or algorithms. However, I disagree. From my own struggles to reading and listening to the struggles of other, when new programmers begin to learn C they often hit a road bump. This road bump is difficult in that it can easily fool one, many believe they can pass through it quickly. This road bump is managing memory in the C language.

Why is it so difficult? Most newcomers to programming begin with a language that does not support memory management on this level. As an example, my first two programming heavy classes that I have taken as part of my ongoing education have focuses on the basics in Java. In java, the virtual machine handles with for the programmer with tools such as the garbage collector. It is out of sight and out of mind. Since that class, the university I am attending has modified the first course by including Python in the mix. Once again, Python does not include this low-level memory management. Others may start in a language such as Javascript or PHP, neither offer this functionality.

The C language is the main language in common usage that has this kind of low-level access. To clear up low level access, you have more freedom in defining how and where stuff is stored in memory. So much access, a programmer gets a glimpse into what is going on down at the hardware level. This becomes a road bump because many do not take their time to learn how memory used, what is really is and how that relates to their operations. Even the most experience programmers mess this up, so it is not restricted to just new programmers. Many bugs occur because of mismanagement of C and it is easy to see how. The C language also does not protect the person writing code like other languages will. This bring great power with using this language, but is has consequences. Something that will be dived into detail is how C allows a programmer to do a lot without a lot of fuss, even access parts of memory that are supposed to be untouched.

# Memory and Variables
We often seem to forget the roles of variables and an easily view them in the wrong light. When we see a variable be defined to a value, such as x = 5, we believe that x truly equals 5. That anywhere x is seen, we can replace with 5. However this is not true. Instead think of a variable as nothing but an address. However, it is just a really pretty looking address that is easy on the eyes. But, keep in mind, this is now how the computer actually sees it. Below is an example of an memory address, and code to show how to view an address in c.


   0x7fff42daca38

```c
   #include <stdio.h>
  
  int main () {

        int x = 5;

        printf("%p\n", &x);

        return 0;
  }
```

Right now, I am not going to focus on the code. It is here for a reader to copy/paste, compile and run.

When the program is executed, a variable is created, then in the following line, the address of the variable is printed out. The computer output this hard to read and understand string of text that is in fact the memory address of variable x. When the compilers see x it is actually seeing 0x7fff42daca38. This memory address is represented as a hexadecimal number (you can tell by the 0x in the beginning). Memory and Arrays

Now that memory has been seen on a single variable level, lets take a look at how this relate to arrays. When is comes to arrays, this is where the most problems can occur, as mentioned earlier, C does not protect the programmer from doing something unintended.

Arrays allows us represent multiple pieces of data that are similar in a somewhat organized fashion. In C, an array of integers can only contain multiple values of integers, a double or float value will not be stored in an array of integers.

First thing to note on arrays from a view from C, and this occurs in many other languages, each index in an array as a set size. How does it determine the set size of the index? It goes based off of the type. Each type takes up a certain amount of bytes. An integer is represented by a certain number of bytes, a long is represented by a certain number of bytes, etc. This is why we have long and int. Long types can store a larger number than an int. Below is a program that can run to demonstrate this, and I also include the output from my machine. Keep in mind, the output can be different depending on the machine this is run on.

```c
  #include <stdio.h>
  
  int main () {

        printf("int: %d\n", sizeof(int));
        printf("char: %d\n", sizeof(char));
        printf("long: %d\n", sizeof(long));
        printf("short: %d\n", sizeof(short));
        printf("float: %d\n", sizeof(float));
        printf("double: %d\n", sizeof(double));

        return 0;
  }
```

  int: 4
  char: 1
  long: 8
  short: 2
  float: 4
  double: 8

Each type has a size. An integer type uses 4 bytes while a char uses 1 byte. Using this, we can gauge the size of an array. An array of integers with 4 elements each taking up 4 bytes of memory results in a total size of 16 bytes of memory.

```c
  #include <stdio.h>
  
  int main () {

        int x[4] = {1, 2, 3, 4};

        printf("Size of array: %d\n", sizeof(x));

        return 0;

  }

  ```

  Size of array: 16

Now, how does this relate to memory management. First, when the value in an index is asked for, it pulls it by using the address where it resides. Every index in an array has an address. It determines this address by two factors, the first as explained above, is the amount of memory each type takes up. So an integer has a size, or think about it as width, of 4 bytes. Now here is the important part, especially when it comes to passing array to functions, the computer calculates the address of an individual index based off of the address of the first index. When an array is passed to a function, each address of each index IS NOT passed. The value passed is the address of the first element. Then to figure out what value is in the nth index, the computer does pointer math, or uses the size of the type and adds that to the address to find an index.

That might sound confusing, but to assist, there is code to go with.

```c
  #include <stdio.h>
  
  int main () {

        int x[4] = {1, 2, 3, 4};

        printf("Size of array: %d\n", sizeof(x));
        printf("Array Address: %p\n", &x);
        printf("First Element (Index 0): %p\n", &x[0]);
        printf("Second Element (Index 1): %p\n", &x[1]);
        printf("Third Element (index 2): %p\n", &x[2]);

        return 0;

  }
  ```

```
  Size of array: 16
  Array Address: 0x7fffda0ea380
  First Element (Index 0): 0x7fffda0ea380
  Second Element (Index 1): 0x7fffda0ea384
  Third Element (index 2): 0x7fffda0ea388
```

Notice that address of the array is the same as the first element, or index, in the array. Also notice that between the different indexes, the memory address is almost the same except for the last character in the stream. Each element differs by 4 in the end; 380, 384, 388.

Here is another interesting trick with C. C does not necessarily care about what the actual value is being stored, it is concerned with what fits.

```c
  #include <stdio.h>
  
  int main () {

        int x = 3.14;
        printf("%d\n", x);

        x = 'c';
        printf("%d\n", x);;

        return 0;
  }
```

```
  3
  99
```

How does this involved a common mistake made by programmers? When an array is passed, it passes the address of the index of the first element. It will them use “memory math” to get the address where the other indexes are stored. When iterating through an array in C, one must be really careful to explicitly define the bounds of the array, otherwise, it will keep iterating even outside of the array. We can define the bounds of the array by using an explicit number if we know an array will only ever by of size n indexes.
