---
date: "2023-02-24"
title: Measuring context switch latency
tags: ["os", "go", "performance"]
---

I wanted to understand measure context switch time between two processes. The standard way to do this is to ping pong some data between two processes. Here I do the transfer using two pipes one for each direction of the transfer

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/time.h>
#include <sys/types.h>

int main()
{
    int f1[2], f2[2];
    int n = 1000000;
    pipe(f1);
    pipe(f2);
    pid_t childPid = fork();
    if (childPid == 0) {
        int i;
        for (i = 0; i < n; i++) {
            read(f1[0], &i, sizeof(i));
            write(f2[1], &i, sizeof(i));
        }
    } else {
        struct timeval tval_before, tval_after, tval_result;
        gettimeofday(&tval_before, NULL);
        int i;
        for (i = 0; i < n; i++) {
            write(f1[1], &i, sizeof(i));
            read(f2[0], &i, sizeof(i));
        }
        gettimeofday(&tval_after, NULL);

        timersub(&tval_after, &tval_before, &tval_result);

        printf("Time elapsed: %ld.%06ld\n",
                (long int)tval_result.tv_sec,
                (long int)tval_result.tv_usec);
    }
    return 0;
}
```

On my m1 macbook air this takes 3 seconds to complete for 1.5 microseconds per context switch. This number is much much smaller than I'd expected. In the past I've some times wondered how much does allocating work to threadpools cost and considering the work done on a threadpool typically involves network or disk access the cost paid for the context switch is pretty low.

Assuming the numbers from this talk still hold [https://www.youtube.com/watch?v=KXuZi9aeGTw](https://www.youtube.com/watch?v=KXuZi9aeGTw) mac os is able to pin the two processes correctly on two processors and this 1.5 microseconds of context switch time is due to the work done by the os scheduler.

Unsurprisingly the golang equivalent of exchanging ints over a channel is blazing fast. This takes only 195 nanoseconds per context switch on my machine.

```go
package main

import (
        "fmt"
        "time"
)

const N = 1000000

func A(c chan int, fin chan int) int {
        var x int
        for i := 0; i < N; i++ {
                c <- 1
                x = <-c
        }
        fin <- 1
        return x
}

func B(c chan int, fin chan int) int {
        var y int
        for i := 0; i < N; i++ {
                y = <-c
                c <- 1
        }
        fin <- 1
        return y
}

func main() {
        c := make(chan int)
        fin := make(chan int, 2)
        st := time.Now()
        go A(c, fin)
        go B(c, fin)
        <-fin
        <-fin
        ed := time.Since(st)
        fmt.Printf("Took %.2f seconds for %d nanoseconds per switch",
                ed.Seconds(), ed.Nanoseconds()/(2*N))
}
```