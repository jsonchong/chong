**Linux** is a clone of the Unix operating system,It is a multiuser,multitasking operating system that runs on wide variety of platforms.

The following diagram shows the main Linux building blocks:

<img src="https://static.packt-cdn.com/products/9781838646554/graphics/assets/9a7b4405-e9e4-431b-b068-0883d1189150.png" alt="img" style="zoom:24%;" />

 Let's describe the layers we see in the diagram:

> 1. On the top layer,there are user applications,process,compilers,and tools.This layer(which runs in a user space)communicates with the Linux kernel(which runs in kernel space)through system calls.
>
> 2. System libraries: These are a set of functions through which an application can interact with the kernel.
>
> 3. Kernel: This component contains the core of the Linux system. Among other things,it has the scheduler,
>
>    Networking,memmory mangement,and filesystems.

Linux has a pretty standard folder structure across the distributions, so knowing this would allow you to easily find programs and install them in the correct place.Let's have a look at it as follows:

1. open a Terminal 
2. Type the command `ls -l /`

The output of the command will contain the following folders:

<img src="https://static.packt-cdn.com/products/9781838646554/graphics/assets/4a845486-dd7f-40b4-a271-fc851692fe1e.png" alt="img" style="zoom:50%;" />

As you can see this folder structure is pretty organized and consistent across all the distributions.Under the hood,the Linux filesystem is quite modular and flexible. 

A shell is a command interpreter that receives commands in an input,redirects them to GNU/Linux, and returns back the output.It is the most common interface between a user and GNU/Linux.



