# **Implementing multi-threading in cups-browsed**
#
### Brief Overview of cups-browsed
cups-browsed is a daemon, which discovers remote printers and makes them locally available by creating local queues for them. It provides many features to the users, such as, clustering of printers, access to printers from legacy servers, filtering of important printers out of many printers, etc. It is part of [cups-filters](https://github.com/OpenPrinting/cups-filters) and is maintained by [Open Printing](https://openprinting.github.io/).

### Scaling Problem
As it discovers printers on the network one-by-one, and then creates queues for them. For small number of printers, it is not that time consuming. But as number of printers increases, time taken to create queues for all the printers increases a lot. From users' perspective, the time taken should be as low as possible.
Please find more details about the time taken to create queues for different numbers of printers [here](https://github.com/mohitmo/Testing).

#### Possible Solution
To optimize cups-browsed, we think that multi-threading should be a solution to this problem. Since cups-browsed is an event-based daemon i.e., it reacts to various events (adding printers, removing printers, jobs created, job cancelled, etc.) appropriately. In the current state, responses to these events are queued up and performed one after the other. In case of many remote printers the work keeps piling up. Most of these tasks are not loading the CPU completely for all the time they are executing, instead they are waiting for CUPS to do something. Thus if we parallelize these responses, it would decrease the time taken by cups-browsed to react to these events.

#### Proposed Design
##### 1. API Used
We will use pthread API ([POSIX Threads](https://en.wikipedia.org/wiki/POSIX_Threads)). It provides support for thread management, thread synchronization (mutexes, read-write locks, condition variables) etc. in C programming language. You can find more details about it [here](https://pubs.opengroup.org/onlinepubs/7908799/xsh/pthread.h.html). 

