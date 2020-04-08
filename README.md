# CPSC/ECE 3200: Introduction to Operating System - Project #6 (Bouns Project)

This project is an optional bonus task. Task A aims to help you understand how to implment system calls and handle timer interrupts on XV6.

Before writing code, you should make sure you have read "Chapter 4: Traps and device drivers" and "Chapter 6: Scheduling" from the xv6 book and studied the corresponding code. Also, you can review your code for project 1 and relevant slides for how to implement system calls on XV6.


# Task A: Alarm

<code>Add an alarm feature to xv6 that periodically alerts a process as it uses CPU time.</code>

## Overview -- Timer Interrupts

Xv6 uses timer interrupts to maintain its clock and to enable it to switch among compute-bound
processes; the `yield()` calls in `usertrap()` and `kerneltrap()` cause this switching. Timer interrupts
come from clock hardware attached to each RISC-V CPU. Xv6 programs this clock hardware
to interrupt each CPU periodically.

In this task, you'll add a feature to xv6 that periodically alerts a process as it uses CPU time. 
This might be useful for compute-bound processes which want to limit how much CPU time they chew up, 
or for processes which want to compute but also want to take some periodic action. 

More generally, you'll be implementing a primitive form of user-level interrupt/fault handlers; 
you could use something similar to handle page faults in the application, for example. 

Your solution is correct if it passes alarmtest and usertests.

## Specification
+ You should add a new `sigalarm(interval, handler)` system call. 

+ If an application calls `sigalarm(n, fn)`, then after every n "ticks" 
  of CPU time that the program consumes, the kernel should cause application 
  function `fn` to be called. 

+ When `fn` returns, the application should resume where it left off. 

+ A tick is a fairly arbitrary unit of time in xv6, determined by how often 
  a hardware timer generates interrupts.

+ Add the file __user/alarmtest.c to__ the Makefile. It won't compile correctly until 
you've added `sigalarm` and `sigreturn` system calls.

+ __alarmtest__ calls `sigalarm(2, periodic)` in __test0__ to ask the kernel to 
  force a call to `periodic()` every 2 ticks, and then spins for a while. 
  You can see the assembly code for __alarmtest__ in __user/alarmtest.asm__, which may be 
  handy for debugging. 
  
+ Your solution is correct when __alarmtest__ produces output like this and usertests also runs correctly:

```
$ alarmtest
test0 start
......................................alarm!
test0 passed
test1 start
..alarm!
..alarm!
..alarm!
.alarm!
..alarm!
..alarm!
..alarm!
..alarm!
..alarm!
..alarm!
test1 passed
$ usertests
...
ALL TESTS PASSED
$
```

## Implementation Notes/Hints

+ The first challenge is to arrange that the handler is invoked 
  when the process's alarm interval expires. 
  You'll need to modify usertrap() in kernel/trap.c so that 
  when a process's alarm interval expires, the process executes the handler. 
  
  How can you do that? You will need to understand how system calls work 
  (i.e., the code in kernel/trampoline.S and kernel/trap.c). 
  For example,  which register contains the user-space instruction address to which system calls return?

  Your solution will be only a few lines of code, but it may be tricky to get it right. 

### __test0: invoke handler__

Get started by modifying the kernel to jump to the alarm handler in user space, 
    which will cause test0 to print "alarm!". Don't worry yet what happens after the "alarm!" output;
    it's OK for now if your program crashes after printing "alarm!". Here are some hints:

+ Modify the Makefile to cause alarmtest.c to be compiled as an xv6 user program.
+ Put the two new function declarations in user/user.h are:
    '''
    int sigalarm(int ticks, void (*handler)());
    int sigreturn(void);
    '''

+ Update __user/usys.pl__ (which generates __user/usys.S__), __kernel/syscall.h__, and __kernel/syscall.c__ to allow
 `alarmtest` to invoke the `sigalarm` and `sigreturn` system calls.

+ For now, your `sys_sigreturn()` should just return zero.

+ Your `sys_sigalarm()` should store the __alarm interval__ and the __pointer to the handler function__ in new fields in the `proc` structure (in __kernel/proc.h__).

+ You'll need to keep track of how many ticks have passed since the last call (or are left until the next call) to a process's alarm handler; you'll need a new field in __struct proc__ for this too. 

+ You can initialize `proc` fields in `allocproc()` in __proc.c__.

+ Every tick, the hardware clock forces an interrupt, which is handled in `usertrap()`; 
you should add some code here.

+ You only want to manipulate a process's alarm ticks if there's a timer interrupt; you want something like

```
  if(which_dev == 2)  ...
```
    
+ Only invoke the alarm function if the process has a timer outstanding. 
Note that the address of the user's alarm function might be 0 (e.g., in __alarmtest.asm__, __periodic__ is at address 0).

+ It will be easier to look at traps with gdb if you tell qemu to use only one CPU, which you can do by running

```
    make CPUS=1 qemu-gdb
```  

+ You've succeeded if alarmtest prints "alarm!".

### test1(): resume interrupted code

Chances are that alarmtest crashes in test0 
or test1 after it prints "alarm!", or 
that alarmtest (eventually) prints __"test1 failed"__, 
or that alarmtest exits without printing "test1 passed". 

To fix this, you must ensure that, 
when the alarm handler is done, control returns to the instruction
 at which the user program was originally interrupted by the timer interrupt. 
 You must ensure that the register contents are restored to 
 the values they held at the time of the interrupt, so that the user program 
 can continue undisturbed after the alarm. 
 
 Finally, you should "re-arm" the alarm counter after 
 each time it goes off, so that the handler is called periodically.

As a starting point, we've made a design decision for you: 
user alarm handlers are required to call the 
sigreturn system call when they have finished. 
Have a look at __periodic__ in alarmtest.c for an example. 
This means that you can add code to usertrap and sys_sigreturn 
that cooperate to cause the user process to resume properly after it has handled the alarm.

Some hints:

+ Your solution will require you to save and restore registers---what registers do you need to save and restore to resume the interrupted code correctly? (Hint: it will be many).
+ Have usertrap save enough state in struct proc when the timer goes off that sigreturn can correctly return to the interrupted user code.
+ Prevent re-entrant calls to the handler----if a handler hasn't returned yet, the kernel shouldn't call it again.

Once you pass `test0` and `test1`, run `usertests` to 
make sure you didn't break any other parts of the kernel.

You can run `./grade-alarm` to see your potential grade for this task.

## Submission
Please following the procedure below in your submission.

1. Switch to your project folder: 

```cd ~/path/to/your-projects-folder```

2. Clean up the folders
    ```
    cd TaskA
    make clean
    ```
3. List files that you have added or modified: ```git status```
4. Stage the new and modified  files in your local repo: ```git add files-you-created-or-modified```
5.  Commit the changes to your local repo: ```git commit -m "Finished Project #6"```
6. Push the changes to the remote repo (i.e. github): ```git push``` Or ```git push origin master```. 
7. Verify all changes have been push into the repo on github.
    + You can run "git status", "git log", and "git show" to the see the changes and commits you have made
    + You can also log into github to see if your changes have been actually pushed into your project repo on github.
8. (Optional but recommended)  For a further validation, you can check out the repo in a separate folder 
    on your computer and then verify that it has all the files for the programs to work correctly.
