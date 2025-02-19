---
layout: default
title: Processes
description: Notes on processes in the Linux kernel.
nav_order: 1
parent: Linux
grand_parent: Operating Systems
permalink: /operating-systems/linux/processes
---

<!-- prettier-ignore-start -->

# Processes
{:.no_toc}

<details closed markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

<!-- prettier-ignore-end -->

# Introduction

A **process** is an executing instance of a **program**, which is object code stored on some media. There can be multiple processes of the same program running at the same time 

A process includes resources like open files, processor state, a memory address space, one or more threads of execution, and a data section containing global variables 

**Threads** (threads of execution) are the parts of a process that contain the program counter, process stack, and processor registers. The kernel schedules threads rather than processes. "Linux does not differentiate between threads and processes. To Linux a thread is just a special kind of process" 

Processes use a virtual processor and virtual memory. The virtual processor gives the illusion that a process is the only process running on a processor, despite potentially thousands of other processes sharing the processor. Virtual memory is another illusion. As far as a process is aware, it has exclusive access to memory, but in reality the kernel is managing memory access.

Threads share memory with each other, but each one has their own virtual processor 

## The lifecycle of a process

A process is created using the `fork()` system call. The new process that's created, the child process, inherits from the parent process that was active when `fork()` was called. The parent process resumes execution after the call to `fork()`, and the child starts execution at the same position. Internally `fork()` uses the `clone()` system call 

The `exec()` system call family "creates a new address space and loads a new program into it" 

The `exit()` system call terminates a process and frees its resources. An exited process becomes a zombie process representing a terminated process until the parent process calls `wait()` or `waitpid()` 

### fork()
A UNIX `fork()` system call

```c
#include <stdio.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[]){
        printf("hello world (pid:%d)\n", (int) getpid());
        // Note that the parent process gets returned the child PID, while the child gets a return value of 0
        int rc = fork();
        if (rc < 0){
                //fork failed
                fprintf(stderr, "for failed\n");
                exit(1);
        }else if (rc == 0){
                // child (new process)
                printf("hello, I am child (pid:%d)\n", (int) getpid());
        }else{
                // parent goes down this path (main)
                printf("hello, I am parent of %d (pid:%d)\n", rc, (int) getpid());
        }
        return 0;
}
```

You should see output similar to the following

```bash
prompt> ./p1
hello world (pid:29146)
hello, I am parent of 29147 (pid:29146)
hello, I am child (pid:29147)
prompt>
```

Assuming we run on a single CPU either the child or parent will begin execution (causing the output to switch). We need to implement `wait()` in order to better control this

### wait()
A UNIX `wait()` system call

Sometimes it is quite useful for a parent to wait for a child process to finish what it has been doing, accomplished with `wait()` or it's more complete sibling `waitpid()`

We will insert the following line at the beginning of the else statement

```c
int rc_wait = wait(NULL);
printf("hello, I am parent of %d (rc_wait:%d) (pid:%d)\n",
            rc, rc_wait, (int) getpid());
```

### exec()
The `exec()` system call is useful when you want to run a program that is different from the calling program. Given the name and some executable `exec()` loads code (and static data) from that executable and overwrites its current code segment (and current static data) with it. The heap and stack and other parts of the memory space of the program are re-initialized

{: .important}
`exec()` does not create a new process, rather, it transforms the currently running program into a different running program. After `exec()` its almost like `fork.c` from above never ran in the child. A successful call to `exec()` never returns

```c
#include <stdio.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

int main(int argc, char *argv[]){
        printf("hello world (pid:%d)\n", (int) getpid());
        int rc = fork();
        if (rc < 0){
                //fork failed
                fprintf(stderr, "for failed\n");
                exit(1);
        }else if (rc == 0){
                // child (new process)
                printf("hello, I am child (pid:%d)\n", (int) getpid());
                char *myargs[3];
                myargs[0] = strdup("wc");   // program: "wc" (word count)
                myargs[1] = strdup("fork.c"); // argument: file to count
                myargs[2] = NULL;           // marks end of array
                execvp(myargs[0], myargs);  // runs word count
                printf("this shouldn’t print out");
        }else{
                // parent goes down this path (main)
               int rc_wait = wait(NULL);
                printf("hello, I am parent of %d (rc_wait:%d) (pid:%d)\n",
                        rc, rc_wait, (int) getpid());       
        }
        return 0;
}
```

### Why `fork()` and `exec()`
The use of these two system calls is essential in how the shell actually functions when doing normal business. You type a command (i.e., the name of an executable program, plus any arguments) into it; in most cases, the shell then figures out where in the file system the executable resides, calls `fork()` to create a new child process to run the command, calls some variant of `exec()` to run the command, and then waits for the command to complete by calling `wait()`. When the child completes, the shell returns from `wait()` and prints out a prompt again, ready for your next command

This allows commands such as `prompt> wc p3.c > newfile.txt` to work. When `fork()` is called, the child process can be changed from the parent process in order for file output to be changed from standard output to the file `newfile.txt`


## The task structure

Internally, a process is referred to as a **task**. A task is represented as a `task_struct` in the kernel. Each `task_struct` is stored in a circular doubly linked list called the **task list** .

A `task_struct` is large, at around 1.7KB on a 32-bit machine (in Linux 2.6). `task_struct` structs are allocated by the slab allocator for efficient object reuse .

Until the 2.6 kernel the `task_struct` was stored at the end of the kernel stack of a process. This made it possible to access the task struct using the stack pointer. Now that the slab allocator manages the `task_struct` there is a `thread_info` struct which performs the same function, either stored at the bottom of the stack for stacks that grow down, or the top of the stack for stacks that grow up .

### Storing the process descriptor

Each process is given a unique process identifier, or **PID**. A PID is a numerical value of type `pid_t` (usually an `int`). The default maximum value is 32,768 for backwards compatibility with older Linux versions, but it can be set up to 4,194,304 by editing the value in /proc/sys/kernel/pid_max. The maximum value is important because it limits the number of processes that can exist at once .

Most kernel code deals with the `task_struct` directly, so it’s important to be able quickly access a process. The currently executing process can be accessed using the `current()` macro. The `current()` macro is architecture specific, because different architectures store the `thread_info` structure at different locations .

Some architectures store a pointer to the current `thread_info` structure in a register, so the `current()` macro on those architectures just returns the value from the register. Register-constrained architectures like x86 store the process on the stack. Accessing it on an x86 is done by masking out the 13 least significant bits of the stack pointer, assuming the stack size is 8KB:

```

movl $-8192, %eax
andl %esp,%eax

```

`current` then deferences the `task` member of `thread_info` to return a pointer to the current `task_struct`:

```c
current_thread_info()->task;
```



### Process state

The `task_struct` `state` field describes the current condition of the process. Each process will be in one of five possible states:

1. `TASK_RUNNING`
2. `TASK_INTERRUPTIBLE`
3. `TASK_UNINTERRUPTABLE`
4. `__TASK_TRACED`
5. `__TASK_STOPPED`

`TASK_RUNNING` means the process is either currently running or on a run queue waiting to run.

`TASK_INTERRUPTIBLE` means the process is waiting for some condition to exist. When the condition exists, the kernel sets the state to `TASK_RUNNING`. The process will awake prematurely if it receives a signal.

`TASK_UNINTERRUPTABLE` is the same as `TASK_INTERUPTIBLE` except the process will not awake prematurely if it receives a signal.

`__TASK_TRACED` means the process is being traced by a program like ptrace.

`__TASK_STOPPED` means the process is no longer being executed. "This occurs when the task receives the `SIGSTOP`, `SIGTSTP`, `SIGTTIN`, or `SIGTTOU` signal or if it receives any signal while it is being debugged" .

The kernel can change a process's state with the `set_task_state()` function:

```c
set_task_state(task, state);
```

The kernel can change the current process state with the `set_current_state()` function:

```c
set_current_state(state)
```



### Process context

Normally a process runs in user space, but when the process executes a system call the processor enters kernel space, where the kernel executes on behalf of the process. At this point the kernel is running in process context (as opposed to interrupt context). In process context, the `current()` macro is valid .

"System calls and exception handlers are well-defined interfaces into the kernel. A process can begin executing in kernel space only through one of these interfaces—all access to the kernel is through these interfaces" .

### The process family tree

All processes are descendants of the `init` process. The `init` process is started by the kernel as the last step in the boot process .

Every process has one parent, and every process has zero or more children. Processes that are children of the same parent are called **sibling processes**. Each `task_struct` has a `parent` field with a reference to the parent, and a `children` field which is a linked list that contains the children .

The `init` task `task_struct` is statically allocated as `init_task` .

When a task is created it’s added to the task list, you can access the previous and next item on the `task_list` with `list_entry()`:

```c
list_entry(task->tasks.next, struct task_struct, tasks);
list_entry(task->tasks.prev, struct task_struct, tasks);
```

You can iterate over the entire task list using the `for_each_process()` macro, although it's an expensive $$O(n)$$ operation .

## Process creation

You create a task using the `fork()` and the `exec()` family calls. `fork()` clones the current task, updating the PID and some other values. `exec()` loads a new executable into the address space and begins executing it .

A naive approach to `fork()` would be to copy all resources for a new process. This is expensive and often unnecessary because many resources are never rewritten and can be shared as read-only. Linux follows copy-on-write where copying of the address space only happens if the data is rewritten. Linux marks pages so that when a page is written to, it’s copied and each process receives a unique page containing the data. This makes process creation fast because the kernel only has to change a few values, rather than copying over a potentially large address space .

`fork()` is implemented using the `clone()` system call. `clone()` accepts flags that tell it what resources should be shared between the parent and child processes. Clone then calls `do_fork()`.

`do_fork()` calls `copy_process()` which then does most of the work. `copy_process()`:

1. Calls `dup_task_struct()` to create a new stack, `thread_info` structure, and `task_struct` for the new process.
2. Ensures child process doesn't exceed the limit on the number of processes for the current user.
3. Clears or resets `task_struct` fields that are unique to a process. Most values in `task_struct` are unchanged.
4. Sets the new process's `task_struct` `state` field to `TASK_UNINTERRUPTABLE` to ensure it doesn’t run yet.
5. Calls `copy_flags()` to update the `flags` member of the new `task_struct`.
6. Assigns a new pid with `alloc_pid()`.
7. Depending on flags passed to `clone()`, `copy_process()` either duplicates or shares resources like open file descriptors.
8. Returns a pointer to the new `task_struct`.

In `do_fork()` the child process is then run before the parent process. In the common case the child calls `exec()` immediately, so running the child first eliminates any overhead from copy-on-write that might occur if the parent ran first .

## Threads

Threads are an abstraction that provide multiple execution threads in the same shared memory address space. Threads enable true parallelism on multiple processor machines.

In the kernel a thread is simply a `task_struct` that shares certain resources, like an address space, with other processes .

Threads are created in the same way as normal tasks, except the `clone()` system call is called with flags for the shared resources:

```c
clone(CLONE_VM, CLONE_FS, CLONE_FILES, CLONE_SIGHAND, 0);
```

This creates a cloned process that shares address space, file system resources, file descriptors, and signal handlers .

### Kernel threads

Kernel threads are processes that exist only in kernel space. "The significant difference between kernel threads and normal processes is that kernel threads do not have an address space". Kernel threads are scheduled and preemptable like normal processes .

Kernel threads can only be created by other kernel threads. They are used for system tasks like the flush task. All new kernel threads are forked from the `kthreadd` kernel process. You can create a kernel thread with the `kthread_create()` function. The thread will be created in an unrunnable state. You can create and start a kernel thread with `kthread_run()`. Once started a kernel thread continues to exist until either it calls `do_exit()` or another thread calls `kthread_stop()` .

## Process termination

When a process dies, the kernel frees the resources owned by the thread and notifies the process's parent that the process has died. A process can terminate itself by calling `exit()` either explicitly, or implicitly by returning from a `main()` function. A process can also terminate in response to a signal or exception that it can’t handle or ignore .

When a process terminates, most of the work is done by `do_exit()`. `do_exit()`:

1. Sets the `PF_EXITING` flag in the `flags` member of the exiting task struct.
2. Calls `del_timer_sync()` to remove kernel timers.
3. Calls `acct_update_integrals()` to write out accounting information if BSD process accounting is enabled.
4. Calls `exit_mm()` to release `mm_struct` held by the process. `mm_struct` is destroyed if no other process is using it.
5. Calls `exit_sem()`. If process is queued waiting for an IPC semaphore, it’s dequeued.
6. Calls `exit_files()` and `exit_fs()` to decrement the usage count of objects related to file descriptors and filesystem data. If either usage reaches zero the object is no longer in use and is destroyed.
7. Sets the tasks exit code, stored in the `task_struct` `exit_code` field.
8. Calls `exit_notify()` to send signals to the parent process. Reparents any of the task's children, and sets the task's exit state to `EXIT_ZOMBIE`.
9. Calls `schedule()` to switch to a new process. Since the process is now unschedulable, it never runs again.
10. After `do_exit()` completes, the `task_struct` exists in a zombie state. When the parent has obtained information about the child, the `task_struct` is freed.


The `wait()` family of system calls are implemented by the `wait4()` system call. `wait()` calls stops the execution of a task until one of its children terminates, and the function returns with the PID of the terminated child that exited

When the `task_struct` is ready to be released, `release_task()` is called. `release_task()`:

1. Detaches the pid and removes the task from the task list.
2. Releases any remaining resources that were used by the process and finalizes statistics.
3. If task was the last member of a thread group and the leader is a zombie, `release_task` notifies the zombie leader's parent.
4. Frees the pages containing the process stack and `thread_info` and deallocates the `task_struct`.



If a child process's parent dies before the child, then the child process needs to be reassigned to another process. This task is called reparenting. The new parent is either another task in the thread group, or the `init` process. The `init` process regularly calls `wait()` on its children to clean up any zombies processes that were assigned to it.


