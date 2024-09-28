---
layout: ../../layouts/post.astro
title: 'Intro to Pointers 3'
author: 'Todd'
pubDate: 2022-09-27
---

# Pointers: Working with functions

So far, in the previous articles, I have used pointers within the main function. For the most part, this is inefficient in the process of writing a program. The only time this would need to be done is when really specific control is needed within the main method. In fact, this goes for functions in general. When working with only local variables, pointers in most cases do not make sense. Variable Scope: local variables

To clear up local variables, local variables live in the block of code in which they are being executed. Another way, if a variable is created in a function, it only lives in that function. Once the function has ran to completion, local variables are dumped out of memory and no longer exist. Try to run the following code:

```c
  #include <stdio.h>

  void variable () {
        int x = 5;
  };

  int main () {

        variable();
        printf("%d\n", x);

        return 0;
  }
```

Here is the output from my own machine:

```
p.c: In function ‘main’:
p.c:9:17: error: ‘x’ undeclared (first use in this function)
  printf("%d\n", x);
                 ^
p.c:9:17: note: each undeclared identifier is reported only once for each function it appears in
```

An error has occurred. The variable x is undeclared in the main function even though we clearly declared it in the function variable. That is because x is declared in the variable function, therefore only lives in the variable function. When variable runs to completion, x is removed from memory.

# Variables being passed around

Okay, so now the super cool program from above has been edited, it compiles the world is going to be alright.

```c
  #include <stdio.h>

  void variable (int x) {
        x = 5;
  };

  int main () {

        int x = 0;
        variable(x);
        printf("%d\n", x);

        return 0;
  }
```

Now, we run the output files and we get this:

     0

So, what happened just now? The variable was created in the main function, passed into the variable where it clearly was reassigned to the value of 5. But yet, the output is showing what x was originally set to.

C works different from other languages, in fact, other languages have this, but their compilers handle this for you typically. When the variable x was passed into the variable function, the function created it’s own copy and worked on that copy, leaving the original in tact and untouched.

# Passing around pointers

The solution to the conundrum in the previous section involves the use of pointers. Instead of passed a variable, there by creating a copy to work on, instead we a pointer should be passes. The advantage is, the address of the variable x will be copied into the function, then using what was saw in the past articles on pointers, the address can be used. Now here is the super cool program.

```c
  #include <stdio.h>

  void variable (int* x) {
        *x = 5;
  };

  int main () {

        int x = 0;
        variable(&x);
        printf("%d\n", x);

        return 0;
  }
```

Now the output:

    5

The program now outputs the correct value. By using the address and working on the value stored at the address, the function can not work on the variable in the main function since it is no longer working on a copy. Instead it is working on the values stored at the address of the original.
