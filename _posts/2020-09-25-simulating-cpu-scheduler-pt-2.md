---
layout: post
title: "CPU Scheduling - Part 2 - Setting Up"
category: blog
---

Let's talk about what our final product would look like. We'll write a C program which takes a type of policy and a list of processes as inputs, and outputs the state of our CPU at any given time. If you've read the [previous article](../simulating-cpu-scheduler), you already know the four types of scheduling policies we're going to implement:

- FIFO (First In First Out)
- SJF (Shortest Job First)
- STCF (Shortest Time to Completion First) and
- RR (Round Robin)

We're storing the list of processes in a file called `processes.dat`. It's a simple text file with a specific format which I am going to discuss next. Here's what an example file looks like:

```python
# pname:pid:duration:arrivaltime
# process 1
P1:13:2:1
# process 2
P2:15:1:2
# process 3
P3:16:1:2
```

Any line starting with the pound (#) sign is a comment, so our code must ignore it. The rest of the lines correspond to the processes. Each line represents a single process. Let's take a close look at the first process:

```
P1:13:2:1
```

There are four things, each separated by a colon (:), to describe this process:

1. Name (`P1` in this case). A combination of alphabets and integers.
2. Id or pid (`13` here). Not really useful in our program, but I've kept it for fun.
3. Duration (`2` for process P1). It's the number of units of time the process takes to finish.
4. Arrival time (`1`). When did the process come? Again, a number of units of time.

*All the numeric quantities are non-negative integers*

How do we run this program? After the compilation, we'll be able to run it like this:

```bash
./main processes.dat FIFO
```

Where `./main` is our program and the next two arguments are the processes file and the policy type (respectively).

## The Output

The output of our program will be the state of CPU from start to the end. Each line in the output gives us the state of the CPU at a particular time. Take the following output for example:

<div id="output-format"></div>
```
1:P1:empty:
2:P1:P2(1),P3(1),:
3:P2:P3(1),:
4:P3:empty:
```

Don't freak out! We'll go through it line by line. The first line is:

```
1:P1:empty:
```

This tells us that at the start (1st unit of time) of our CPU execution, a process P1 has arrived and needs to be run. Since there is no other process running, the CPU runs P1. Also, P1 was the only process which came that early, so the waiting line is empty. We call this waiting line a ready state or a ready queue. In summary, each output line has the following format:

```
Time:Currently Running Process:Ready State:
```

Next, we have:

```
2:P1:P2(1),P3(1),:
```

One unit of time has passed by and ohh, two more processes (P2 and P3) have arrived but seems like the previous process has still not finished since it's duration was 2 units of time (we got that from the processes file). So what do we do? we put both incoming processes in the ready queue and as soon as the currently running process P1 stops, we  run the next ready process.

*The number in paranthesis **(1)** is the duration of process*

The CPU is now at the 3rd unit of time:

```
3:P2:P3(1),:
```

> While reading this article, you must have read the phrase "units of time" and may be wondering why am I not using a defined unit such as seconds, milliseconds or microseconds. That's because we're leaving it your CPU. For us, 1 unit of time is the time your CPU takes to execute an instruction.

P1 has finished and now it's gone. It's time for P2 to run. So we remove it from the ready queue and put it in the running state. It runs for 1 unit of time and finally the last process P3 is run for the same amount of time:

```
4:P3:empty:
```

*The ready queue is empty so we wrote **empty** in the third column.*

At this point, we have some idea of the structure of our scheduling program. We read the file given by the user for the list of processes and store them in some format. Then we run the policy also choosen by the user. Pretty simple, right? Let's write the main function. I've added some comments as a nice person:

```c
#include <stdio.h>
#include <string.h>

int main (int argc, char * argv[]){
    // Get the inputs (filename and policy) from command-line arguments
    char * filename = argv[1];
    char * policy = argv[2];

    // Some error-checking to make sure we get the valid inputs
    if (argc != 3 || (
        strcmp(policy, "FIFO") &&
        strcmp(policy, "SJF") &&
        strcmp(policy, "STCF") &&
        strcmp(policy, "RR")
    )) {
        fprintf(stderr, "Error. Usage: ./mysched filename POLICY\nwhere POLICY can be one of the following strings:\nFIFO\nSFJ\nSTCF\nRR\n");
        return 1;
    }

    // Gotta read that file now
    FILE *fp;
    fp = fopen(filename, "r");

    // Some error-handling
    if (fp == NULL) {
        fprintf(stderr, "Error. Unable to open file %s\n", filename);
        return 1;
    }

    // The file is now ready to be read
    // We will read through it and put them in a queue

    // Queue size (number of processes) will be
    // updated by the 'read_processes' function
    int queue_size = 0;

    // Read the file, put each process in the queue
    // and get the head of the queue
    struct node *head = read_processes(fp, &queue_size);

    // It's good to sort the processes by their arrival time
    // just in case they're not in correct order
    sort_by_arrival(&head, queue_size);

    // Run the policy on the processes queue
    run_scheduler(head, policy);

    // Close the file
    fclose(fp);
    return 0;
}

```

That's a lot of code thrown at you right away! But read through it, I know you'll get most of it. The `read_processes` function returns a pointer to a `struct node`. It looks like this:

```c
struct node {
    struct process *p;
    struct node *next;
};
```

Each node of the queue holds two things: a pointer to the process and a pointer to the next node in the queue (if it exists). A process is another `struct` holding four things. Guess them before looking at:

<div id="process-structure"></div>
```c
struct process {
    char* name;
    int pid; // Process Id
    int ttc; // Time To Complete
    int toa; // Time Of Arrival
};
```

Let's implement the `read_processes` function:

```c
struct node* read_processes(FILE* fp, int * queue_size) {
    struct node *head = NULL;

    // Get the file size in bytes, we will use this as an estimate
    // memory allocation for pid, ttc and toa
    int filesz = get_file_size(fp);

    // Get the first character in the file
    char c = getc(fp);

    // Until we don't reach the end of file, read the characters
    while (c != EOF) {
        bool iscomment = false;
        if (c == '#') iscomment = true;

        // Read each line
        while (c != '\n') {

            // Don't do anything if it's a comment
            if (!iscomment) {

                // We're only allowing 10 characters for the process name
                char name[10];

                // filesz is a safe size to pick (it may even be too much for a single info)
                char pid[filesz];
                char ttc[filesz];
                char toa[filesz];
                
                // Read each character of a line and correctly assign
                // values to name, pid, ttc and toa.

                // ... excluded code for clarity

                // Then enqueue the process using these 4 things

                enqueue(&head, name, atoi(pid), atoi(ttc), atoi(toa));
                (*queue_size)++;
            }
            if (iscomment)
                c = getc(fp);
        }
        
        c = getc(fp);
    }
    return head;
}

```

If you squint hard enough, you'll see that we did not create any queue separately. Or in other words, there is no separate structure defined for a queue. That's because we can simply create a `node` and chain it with subsequent ones using the `next` pointer. That'll be our queue! We can use the `enqueue` function to add a new node (containing a process) at the end of the chain. I am not going to show the implementation of the functions `get_file_size` or `enqueue`, but you can find the complete [source code on Github](https://github.com/FarooqAR/sim_schedular/).

In the [next article](../simulating-cpu-scheduler-pt-3), we'll implement the four policies. See you there!

