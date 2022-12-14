---
layout: post
title:  "NodeJS: Schedule callback function exceeding the maximum allowed delay"
date:   2022-9-6 12:11:28 +0800
categories: NodeJS
---

### **setTimeout(): The task scheduler in NodeJS**

`setTimout()` in NodeJS ([documentation here](https://nodejs.org/dist/latest-v16.x/docs/api/timers.html)) is an API that allows programmers to schedule functions to be run in the future with some specified delay in time. There are at most three arguments programmers can pass in when invoking it. The first argument is the actual function to be run while the second argument is the desired delay after which the function should be executed. For example,

&nbsp;

```typescript
setTimeout(() => { 
  console.log("this will be executed after 1000 miliseconds"); 
}, 1000);
```

&nbsp;

The conosle will only display the text after at least 1000 miliseconds (or 1 second). Under the hood, the tasks seheduled by ```setTimeout()``` will be handled by [Event Loop](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/). There are already plenty of discussions on the mechanism of Event Loop and how NodeJS uses Event Loop to execute operations in an aysnc fashion so that the main thread does not block. It is worth pointing out that NodeJS will not execute the scheduled task at the exact time computed by some current timestamp offset by the delay. The only guarantee is that we will at least need to wait for the delay to elapse before the task can be done. There will be several other operations Event Loop needs to carry out in the same loop, so the exact time at which our scheduled task is run fairly indeterministic.

&nbsp;

### **The maximum allowed delay in `setTimeout()`**

In the documentation, it is mentioned that if the delay variable contains a number more than `2147483647`, it will be set to 1. The magic number itself is the maxium value of a 32-bit signed integer data type. So the natural question becomes: what if we need to schedule a task longer than that limit? Well, to many programmers this might not matter too much if all the scheduling tasks are within the boundary. When converted into days, the maximum allowed delay is between 24 and 25 days, so that seems pretty long already right? But there might still be some use cases in scheduling a task over that time frame. For example, there might be some background operations on filesystem or logs that only need to be done once a month.

&nbsp;

The simplest solution is just to hold on to that task for a bit longer before puting it onto Node’s timer queue. For example, you could try to compute the difference between the maximum allowed delay and the target delay and set a timer on the call to `setTimeout()` itself based on the time difference. For example,

&nbsp;

```typescript
/* Function (task) to execute in the future. */ 
function some_function_too_far_away() { /*...*/ }  
/* Suppose we just want to schedule a task that happens 
   to be 10 seconds after the maxium allowed delay. 
   Remember, all these numbers are in miliseconds. */ 
const MAX_DELAY = 2147483647; 
const target_delay = MAX_DELAY + 10000; 
const diff = target_delay - MAX_DELAY;  
/* Nested calls of setTimeout where the inner call is the 
   one we put the real task on Node's timer queue. */ 
setTimeout(() => {   
  setTimeout(() => {     
    some_function_too_far_away();   
  }, target_delay - diff); 
}, diff);
```

&nbsp;

It seems to work, right? But what if we want to schedule a job for which the difference between the maximum allowed delay and its desired delay is again greater than the limit? For instance, if we try to schedule a task 50 days from now, we would need to have two calls of `setTimeout()` in `MAX_DELAY` so that we can wait for long enough to put the real work onto Node's timer queue. This will become somewhat harder to manage. Also, if there are multiple tasks with the same issues, monitoring these tasks will become quite unwieldy.

&nbsp;

```typescript
/* Function (task) to execute in the future. */
function some_function_even_one_max_delay_is_not_enough() { 
  /*...*/ 
}
const MAX_DELAY = 2147483647;
const target_delay = MAX_DELAY + MAX_DELAY + 10000;
setTimeout(() => {
  setTimeout(() => {
    setTimeout(() => {
      some_function_even_one_max_delay_is_not_enough();
    }, target_delay - 2 * MAX_DELAY);
  }, MAX_DELAY)
}, MAX_DELAY);
```

&nbsp;

---

&nbsp;

### **A solution to the maximum delay limit**

I recently wrote a Typescript library to abstract away the complication of managing those unschedulable tasks. The library[node-jobs-scheduler](https://www.npmjs.com/package/node-jobs-scheduler) features a scheduler that provides an easy way to manage scheduling tasks with delays not bounded by the limit imposed by `setTimeout()`. The scheduler also acts as a center store for hosting the tasks and provides an unified suite of APIs to programmers. The scheduler exposes APIs with more descriptive function names such as `schedule_in_milisec()`, `schedule_in_secs()`, and `schedule_at_date()` for the convenience of the users. When users trying to schedule tasks that exceed the maxium delay, the tasks will be put onto the scheduler's internal `pending_queue` until they become schedulable to be enqueued in Node's timer queue. The scheduler also provides a way for users to remove the tasks already in Node's timer queue or those still not schedulable (i.e. in the `pending_queue`). The details of handling all these mundane operations are hidden from the users of the library.

&nbsp;

### **Using the node-jobs-scheduler**

The library contains `class Scheduler` in `Scheduler.ts` which wraps the above-mentionedfunctionalities under one roof. In most cases, I presume only one instance of `class Scheduler` should be initialized. But in a multi-user application where each user is given his/her own unique scheduler, constructing more than one instance might be appropriate as long as the application does not mix up all those scheduler instances. The initialization can be done when the Node runtime starts up. For example, many Node applications will define either a `server.ts` or `server.js` to service HTTP requests from the client. The scheduler can be constructed by code similar to the following

&nbsp;

```typescript
const scheduler = new Scheduler();
// more initiliaztion code
http.createServer(function (req, res)) {
  // Some code here...
}.listen(PORT);
```

&nbsp;

### **Implementation detail of the libray**

Within `class Scheduler`, I have defined several attributes to manage the lifecycles of the tasks, whether they are schedulable or not based from the standpoint of Node's timer queue. The attributes of `class Scheduler` looks like the following.

&nbsp;

```typescript
class Scheduler {   
  _id_seed = 1;                         /* The unique id for each
                                           job. */   
  _jobs_done = 0;                       /* Keep track number of jobs
                                           done. */
  _jobs_cancelled = 0;                  /* Keep track number of jobs 
                                           cancelled. */   
  _pending_jobs_cancelled = 0;          /* Keep track number of
                                           pending jobs cancelled.*/
  _pending_queue: PriorityQueue         /* Jobs not yet scheduled.*/
  _scheduled_jobs: Map<number, Job>     /* All jobs scheduled but                                             
                                           not yet executed. */   
  _pending_jobs: Map<number, PendingJob>/* All jobs in pending. */   
  _max_delay: number                    /* The maximum allowed delay 
                                           to enqueue in Node's 
                                           timer module. Any job       
                                           exceeding this limit will 
                                           be placed in 
                                           _PENDING_QUEUE first. */   
  _pending_job_handle?: NodeJS.Timeout    
// ...more code 
}
```

&nbsp;

I chose to keep tasks already in Node’s timer queue and those still unschedulable in separate data structures (i.e. `_pending_queue`, `_pending_jobs` and `_scheduled_jobs`). The design choice was based on the characteristics of different task categories. For tasks already on Node's timer's queue, we might need to retrieve the object given an unique id so that the task can be cancelled. As a result, the most suitable data structure for this purpose is Map where key is `id` and value is the `job` object. On the other hands, for the pending tasks, in addition to using a different `Map` to store these pending tasks, I also define a priority queue data structure to store them. The priority queue (i.e. min-heap) is a binary tree data structure maintaining an invariant that the root of the tree has the smallest key value. Moreover, any subtree also needs to maintain the same property. In short, no descenants (direct child nodes, grandchild nodes, and etc) of a node should have keys strictly less than the of the node itself. For our `pending_jobs`, the appropriate key value should be the expected execution time such that the earliest should be always be at the root of the tree.

&nbsp;

When scheduling a task, the delay given by the programmers is actually a relative concept in time. When a log displays there is a job to be done 30 days later, it is important to learn when that statement occurs so that we can estimate the time at which the job is expected to get done. Since jobs are scheduled at different time (i.e. different invokation time of `setTimeout()`), we need to make sure the tasks scheduled time are comparable in some absolute terms. As a result, before placing pending works onto the priority queue, the delays are converted into the same basis by invoking a call to `getTime()` in `Date` data type. As a result, the first item on the queue will be the earliest task to be scheduled based on the computed key values. Then, the scheduler will at appropriate time peek at top of the queue and determine whether the earliest task becomes schedulable at the time of the peeking. If so, the scheduler will pop off the earliest task and any other schedulable tasks currently in the queue by keeping peeking and poping off the next earliest task. The logic handling this portion of the scheduler contains a lot more involved implementation and requires careful thinking but the general structure of the code looks like the following.

&nbsp;

```typescript
try_schedule_pending_job(delay_ofs: number): NodeJS.Timeout {     
  return setTimeout(() => {     
    let earliest_job = pending_queue.min_node() as PendingJob;     
    let delay_after_ofs = earliest_job.timestamp
                          - current_timestamp;      
    while (delay_after_ofs <= MAX_DELAY) {       
      // pops off the min node and enqueu the task onto Node's timer 
      // queue.              
      // calculate the next earliest job and its delay offset.     
    }   
  }, delay_ofs); 
}
```

&nbsp;

There is one more caveat. Even if we cannot schedule any task onto Node’s timer queue or if there is still any pending job left in the pending queue, we will need to set up a timer to remind the scheduler to check again so that we do not leave those tasks permanently in pending. This is done after the while loop exits.

&nbsp;

```typescript
if (pending_queue.heap_size() > 0) {   
  const next_earliest_job = pending_queue.min_node() as PendingJob;    
  const next_delay = next_earliest_job.timestamp - cur_timestamp;   
  const next_delay_ofs = Math.min(next_dealy - MAX_DEALY, 
                                  MAX_DELAY);      
  
  // Clear the scheduling of the pending queue since the scheduling 
  // will be set to next_delay_ofs.   
  // ...    
  try_schedule_pending_job(next_delay_ofs); 
} else {   
  // Clear scheduling of the pending queue. 
}
```

&nbsp;

### **Some more notes about priority queue**

The priority queue in this library is implemented from scratch without using any dependency. However, to make sure the implementation works, I have done some testing on the varaiable APIs provided in `class PriorityQueue`. (In fact, there are also testings in other parts of the library that contain critical code.) Although these test cases are by no means comprehensive, they should give the user of the library some confidence that the algorithm does do its job. Also, I did not test on the running time of the implementation. However, based on reasoning about the code, the `extract_min()` should be O(log(n)) as advertised by the algorithm's design. The recursive part of `extract_min()` comes from a call to `heapify()`. Even though `heapify()` itself is a recursive function, it will not need to compare a node with every other node in the heap. Instead, once it is determined that a child node (left or right) should be promoted to a subtree root, `heapify()` only recurses on that part of the subtree while ignore the other part and the process stays the same at every level of the tree. That observation should establish a running time of O(log(n)).

&nbsp;

---

&nbsp;

*Originally published at* [https://github.com/mikeliaohm/Node-Jobs](https://github.com/mikeliaohm/Node-Jobs)
