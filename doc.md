# **Implementing multi-threading in cups-browsed**



### Brief Overview of cups-browsed

cups-browsed is a daemon, which discovers remote printers and makes them locally available by creating local queues for them. It provides many features to the users, such as, clustering of printers, access to printers from legacy servers, filtering of important printers out of many printers, etc. It is part of [cups-filters](https://github.com/OpenPrinting/cups-filters) and is maintained by [Open Printing](https://openprinting.github.io/).

### Scaling Problem

As it discovers printers on the network one-by-one, and then creates queues for them. For small number of printers, it is not that time consuming. But as number of printers increases, time taken to create queues for all the printers increases a lot. From users' perspective, the time taken should be as low as possible.
Please find more details about the time taken to create queues for different numbers of printers [here](https://github.com/mohitmo/Testing).

### Possible Solution

To optimise cups-browsed, we think that multi-threading should be a solution to this problem. Since cups-browsed is an event-based daemon i.e., it reacts to various events (adding printers, removing printers, jobs created, job cancelled, etc.) appropriately. In the current state, responses to these events are queued up and performed one after the other. In case of many remote printers the work keeps piling up. Most of these tasks are not loading the CPU completely for all the time they are executing, instead they are waiting for CUPS to do something. Thus if we parallelize these responses, it would decrease the time taken by cups-browsed to react to these events.

### Proposed Design

#### 1. API Used

We will use pthread API ([POSIX Threads](https://en.wikipedia.org/wiki/POSIX_Threads)). It provides support for thread management, thread synchronization (mutexes, read-write locks, condition variables) etc. in C programming language. You can find more details about it [here](https://pubs.opengroup.org/onlinepubs/7908799/xsh/pthread.h.html).

#### 2. Events to be parallelized

#### 2.1 Printer Added

We discover printers via [Avahi](https://www.avahi.org/), which facilitates service discovery on a local network via the DNS-SD/mDNS protocol. As the daemon starts, it creates AvahiClient, which creates service browsers for us, to browse for services of specific types for us. Here we create browsers for service types `_ipp._tcp` and `_ipps._tcp`. With each service discovered a callback function `resolve_callback()` is called, it examines the TXT record of the printer, and further calls `examine_discovered_printer_record()`, which calls `matched_filters()` to check for `BrowseFilter` line match in cups-browsed.conf. Also `examine_discovered_printer_record()` checks whether we have a queue for that printer already, if yes, it correctly update the entry, else calls `create_remote_printer_entry()` to add the discovered printer to the list of `remote_printers`. To actually create the printer queue, `recheck_timer()` updates when to call `update_cups_queues()`. 

<p align="center">
  <img src="https://docs.google.com/drawings/d/e/2PACX-1vSfBUXcKllh4t-eU9Y-QEoKBuC2ZecyoaBLPFs5GNC8rTiqEMPGWpJcFKmKUwVyw_gg_boXc3YGxnZg/pub?w=890&h=589" alt="Flow Diagram"/>
</p>



**How will we parallelize this event**

To parallelize queue creation for a printer, we will create threads from `browse_callback()`, so that for every discovered instance of a printer `resolve_callback()` runs in a separate thread. This means that for each instance of a printer, its call of `resolve_callback()`, `examine_discovered_printer_record()`, `matched_filters()` and `create_remote_printer_entry()` will run in a separate thread from other instances.

Since `create_remote_printer_entry()` calls `recheck_timer()` to update when to call `update_cups_queues()`  from main, we will also make `update_cups_queues()` to run in a separate thread.

#### 2.2 Printer Deleted

Printer deletion is relatively easy than addition. For each service removed from the network `browse_callback()` is called and it finds that particular instance in the list of printers and deletes that instance. If no discovered instances are left for the printer, it calls `remove_printer_entry()`. `remove_printer_entry()` correctly identifies that whether the printer to be deleted has a slave (checks whether it is a master of some cluster of printers), if it has slave, it makes that master of the printer. This is done to make sure that we do not delete that clusters' queue. `remove_printer_entry()` also update the status and timeout value of the printer to be deleted, so that when next time `update_cups_queues` is called, its queue gets removed.  

**How will we parallelize this event**

To parallelize it, we will take out the code in `browse_callback()` and create a separate function. That function will delete the instance and will call `remove_printer_entry()` when necessary. We will create a separate thread from `browse_callback()` for our new function.

*Since printer-deletion is not much time consuming event, it could happen that creating separate threads for each service will take more time, as there is significant overhead for creating threads and using locks.

#### 3. How we will handle synchronization

Thread synchronization is defined as a mechanism which ensures that two or more concurrent [processes](https://en.wikipedia.org/wiki/Process_(computer_science)) or [threads](https://en.wikipedia.org/wiki/Thread_(computer_science)) do not simultaneously execute some particular program segment known as [critical section](https://en.wikipedia.org/wiki/Critical_section). Here critical section will be the global variables, i.e. we don't want that two or more threads simultaneously try to update the global variables, and if this happens integrity of the data will be compromised.

To avoid all this we need to make sure that while updating some global variable, only one thread is doing that update and pthread API provides many useful tools, e.g. mutexes (`pthread_mutex_t`) and read-write Locks (`pthread_rwlock_t`). To prevent other thread from updating these variables, a thread can lock the mutex/read-write locks and when the update is complete unlock them. If a thread tries to lock an already locked mutex, it gets blocked until that mutex is unlocked by the thread that locked the mutex. As soon as it gets unlocked other thread can lock it and do the update. So we will setup these mutexes/read-write locks and whenever a function running in seperate thread tries to update the global variables.
