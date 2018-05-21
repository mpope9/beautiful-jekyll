---
layout: post
title: Jumping VS Function Calling
maintitle: Jumping VS Function Calling
tags: [c, assemby, may, 2018]
---

![]()
Frequently I have heard to avoid `goto` because code will become an unreadable mess.  This has been burned into my brain, I have never considered using a `goto`.  Until that foggy, grey day.  Some...spectre attached itsself to me.  It was summoned from the incantation I foolishly muttered, deep within the GCC documentation.  

I recall that `goto` is fine if trying to break from a nested loop, but if one was to run code based on conditions things fall apart quite quickly.  This is why loops were created, so the `goto` can be avoided, so said Dijkstra, so shall it be<sup>[[2]](https://www.cs.utexas.edu/users/EWD/ewd02xx/EWD215.pdf)</sup>  It soon rested in legends.  Yet I was interested on just how unreadable using `goto` can become.  What is the most inapropriate use of the keyowrd?  `goto`'s syntax officially is `goto label`<sup>[[1]](http://en.cppreference.com/w/cpp/language/goto)</sup>.  This is all good, but what type is a label?  

Well according to GCC's docs, a label's address can be stored.  Infact, they provide an example of `goto` that can take a dereferenced  `void *`<sup>[[3]](https://gcc.gnu.org/onlinedocs/gcc/Labels-as-Values.html)</sup>.  Traditionally this would be the address of a label, obtained with the `&&` operator.  Taking this into account, we can think of `goto` as a keyword that goes to a memory address.  Note: this is only for gcc!  Unfortunatly, clang throws errors for things we do later!  But compiler specific behavior is interesting, so why not explore?

See this example here:

```
// goto.c

int main() {
   void * label_test_pointer;

label_test:
   return 0;
   
   label_test_pointer = &&label_test;

   goto *label_test_pointer;

   return 0;
}
```

Well, using `gcc -S goto.c` we can see this assembly code (onlt the juicy parts are included).  

```
// goto.s

.LFB0:
        .cfi_startproc
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
.L2:
        movl    $0, %eax
        popq    %rbp
        .cfi_def_cfa 7, 8
        ret 
        .cfi_endproc
```


What we are doing is using the unary `&&` operator to store the address of the label into a `void *`.  This is interesting, this speaks to me.  I hear its voice as I sleep.  It says, 

	"I have dark secrets.  I know that you're hungry."

I mutter nonsense in a cold sweat.  I pant heavily.  My head rolls restlessly on my damp pillow.

	"Function pointer"

It whispers into my ear, cold breath tickles the nape of my neck.  I must obey.

```
// jumper.c

#include <stdio.h>

void printer();

int main()
{
   printf("In main\n");

   void (*function_pointer)() = &printer;

   goto *function_pointer;

   return 0;
}

void printer()
{
   int (*main_pointer)() = &main;

   printf("In jumper\n");

   goto *main_pointer;
}
```

And it's assembly, stripped of it's frivioulus includes and printing.

```
// jumper.s

.LFB0:
   .cfi_startproc
   pushq %rbp
   .cfi_def_cfa_offset 16
   .cfi_offset 6, -16
   movq  %rsp, %rbp
   .cfi_def_cfa_register 6
   leaq  printer(%rip), %rax
   movq  %rax, -8(%rbp)
   jmp   *-8(%rbp)
   .cfi_endproc
.LFE0:
   .size main, .-main
   .globl   printer
   .type printer, @function
printer:
.LFB1:
   .cfi_startproc
   pushq %rbp
   .cfi_def_cfa_offset 16
   .cfi_offset 6, -16
   movq  %rsp, %rbp
   .cfi_def_cfa_register 6
   leaq  main(%rip), %rax
   movq  %rax, -8(%rbp)
   jmp   *-8(%rbp)
   .cfi_endproc
```

So what we're looking at here is that we can store a function pointer to a fuction, `printer`, in main and a function pointer in that `printer` function to our `main`.  Then we can goto each function, for as long as we want.  What is interesting is that this runs in an infinite loop, and does not segfault.  Now what we see above that that we're storing the function pointer with a `movq`, and then we are jumping to what was moved with the `jmp` function.  We are not constrained by the stack.  When we jump, we disregard everything.

When we call normal functions, thats not the case.  Gaze upon this weak, overflowing cousin of our jumping code.  The ghost laughs in my ear, long nails scrape my back.

```
// functions.c

#include <stdio.h>

void printer();

int main()
{
   printf("In main\n");

   printer();

   return 0;
}

void printer()
{
   printf("In jumper\n");

   main();
}

```

And here is it, striped naked of it's communication abilities.

```
// functions.s

main:
.LFB0:
   .cfi_startproc
   pushq %rbp
   .cfi_def_cfa_offset 16
   .cfi_offset 6, -16
   movq  %rsp, %rbp
   .cfi_def_cfa_register 6
   movl  $0, %eax
   call  printer
   movl  $0, %eax
   popq  %rbp
   .cfi_def_cfa 7, 8
   ret
   .cfi_endproc
.LFE0:
   .size main, .-main
   .globl   printer
   .type printer, @function
printer:
.LFB1:
   .cfi_startproc
   pushq %rbp
   .cfi_def_cfa_offset 16
   .cfi_offset 6, -16
   movq  %rsp, %rbp
   .cfi_def_cfa_register 6
   movl  $0, %eax
   call  main
   nop
   popq  %rbp
   .cfi_def_cfa 7, 8
   ret
   .cfi_endproc
```

What we see here is the standard `call` to the printer function.  What that does is pile the stack frames onto the stack, like bodies of the vanquished onto a pike.  They soon pile to high and, oh no, we get an overflow.  So when we `jmp` we are not adding anything to the stack, which is not great for returning from main.  We lose the return location, and smash our stack if we try to stop execution in any way.

What I believe is needed is this line of assembly missing from the jumper.c file, that is called right before the `call` assembly in functions.

`movl  $0, %eax`

After some light googling it looks like this is the assembly funcition to push `main`'s return value onto the stack, but I can't seem to get the inline assembly to actually work.  So for now I'll say that is isn't very feasible to use this at all, and other compilers don't even support it.  

It's true what they say about `goto` adding unneeded complexity, see the example I've posed bellow that prints if the current second is odd or even, then ends after 10 seconds.  It segfaults because of stack smashing unfortunatly.  But if we concider the loop in c, we see that it generates a `jmp` instruction.  So a loop might be better in all cases.  

```
// time_jumper.c

#include "time.h"
#include "stdio.h"
#include "unistd.h"
#include "stdlib.h"

unsigned int end_time;
unsigned int started = 0;

void if_even();
void if_odd();
int main();

void (*if_even_pointer)() = &if_even;
void (*if_odd_pointer)() = &if_odd;
int (*main_pointer)() = &main;

int main()
{
   unsigned long start_time = (unsigned long) time(NULL);
   //unsigned long end_time = (unsigned long) start_time + 2;

   if (started == 0) {
      started = 1;
      end_time = (unsigned int) time(NULL) + 2;
   }

   __asm__{"%0, %ea\n"};

   if ((unsigned long) time(NULL) >= end_time)
   {
      printf("Ending! \n");
      return 0;
   }
   else if ((unsigned long) time(NULL) % 2) {
      goto *if_even_pointer;
   }
   else {
      goto *if_odd_pointer;
   }
   
   exit(0);
}

void if_even() {

   printf("This second was even.\n");

   sleep(1);

   goto *main_pointer;
}

void if_odd() {

s second was odd.\n");

   sleep(1);

   goto *main_pointer;
}
```

While that example was certainly messy, I believe we can go deeper.  The phantom again appears before me.  The world fades away.  I am in a pure white corridor.  Florecent lights flicker above me.  I hear:

	"Arrays"

The voice is of ice, goosebumps rise on my skin.

My eyes snap open.  My trance is broken.  A fire rages in my chest.

```
// arrays.c 

#include <stdio.h>

void i_am_who();
void i_think_i_am();
void because_i();
void am_who();

void (*array[4])();

int i_think = 0;

int main()
{
   array[0] = &i_am_who;
   array[1] = &i_think_i_am;
   array[2] = &because_i;
   array[3] = &am_who;

   goto *(array[0]);

   return 0;
}

void i_am_who()
{
   printf("i am who ");
   
   goto *(array[1]);
}

void i_think_i_am()
{
   printf("i think i am ");
   
   if (i_think) {
      i_think = 0;
      printf("\n");
      goto *(array[0]);
   }
   goto *(array[2]);
}

void because_i()
{
   printf("because i ");
   
   goto *(array[3]);
}

void am_who()
{
   printf("am who ");

   i_think = 1;
   
   goto *(array[1]);
}
```

It is extinguished.  I feel the presence of the phantom depart.  It is unbound, I am free.  I collapse upon the ground.

Keep it real,

Matthew
