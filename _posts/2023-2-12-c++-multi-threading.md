---
layout: post
title:  "C++ multi-threading (and the synchronization) in a Qt App"
date:   2023-2-12 10:11:28 +0800
categories: C++ Qt multithreading
---

This post discusses the multi-threading and synchronization implementation of a [Qt application](https://mikeliaohm.github.io/c++/qt/gui/multithreading/2023/02/10/export-photo-albums.html) to export albums in MacOS' `Photos` app. Check out the link to the previous blog mentioned above if you are interested in the overall project layout and implementation details of the GUI. This post instead goes into the implementation detail of the multi-threading part of the application.

<!-- omit in toc -->
## Table of Contents

- [Introduction to Multi-threading and Synchronization](#introduction-to-multi-threading-and-synchronization)
- [The Use of Multi-threading in Album Exporter](#the-use-of-multi-threading-in-album-exporter)
  - [Data Structures](#data-structures)
    - [mainwindow.cpp](#mainwindowcpp)
    - [Task.h: **struct Task**](#taskh-struct-task)
    - [Task.h: **TaskManager::tasks**](#taskh-taskmanagertasks)
    - [Task.h: **TaskManager::buk\_to\_process**](#taskh-taskmanagerbuk_to_process)
  - [Spawn the worker threads](#spawn-the-worker-threads)
    - [mainwindow.cpp: **act\_on\_next**](#mainwindowcpp-act_on_next)
    - [TaskExe.cpp: **pick\_up\_bucket**](#taskexecpp-pick_up_bucket)
  - [Use QProgressDialog to Indicate the Progress](#use-qprogressdialog-to-indicate-the-progress)
    - [Task.cpp: **update\_progress**](#taskcpp-update_progress)
  - [Communication between main thread and worker threads](#communication-between-main-thread-and-worker-threads)
    - [TaskExe.cpp: **complete\_task**](#taskexecpp-complete_task)
    - [mainwindow.cpp: **update\_progress**](#mainwindowcpp-update_progress)
- [Conclusion](#conclusion)

## Introduction to Multi-threading and Synchronization

Multi-threading allows the program to run tasks concurrently and in today's multi-core processors the operating system can schedule multiple threads on different cores to execute the tasks at the same time, making the use of multi-threading in a modern program an efficient way to boost the performance of the application. One way to use multi-threading is to spawn multiple threads to execute a single task simultaneously and the code needs to be set up to break the task into chunks that can be executed by these different threads. Therefore, these threads will most likely need to access the same variable (memory) to see or update the state of the progress of the task being executed. Therefore, synchronization tools are employed so that we could fine tune the behavior when multiple threads trying to access the same variable (memory) at the same time.

There are essentially two types of memory access, i.e. reading of and writing to the memory. In the case where all the threads are only reading the memory, synchronization is not required since the memory is not changed at all. However, in the case where there exists at least one memory writing thread, synchronization is needed so that the updating done by the writing thread can be reflected as another thread (which could be a writer or reader) tries to read or write the same memory. Without synchronization, the behavior of the execution is pretty much undefined and we run into an issue of a `data race`.

## The Use of Multi-threading in Album Exporter

### Data Structures

In my application, multi-threading is used when the program exports the photos in the albums. Since the exporting could involve thousands of photos, it seems ideal to break the photos to be exported into buckets and create multiple worker threads to export these photos concurrently. Here I define a bucket an unit of work for a thread and each bucket contains some fixed number of photos and the fixed number is defined as `TASK_BUCKET_SIZE` in `mainwindow.cpp`.

#### mainwindow.cpp

```C++
#define TASK_BUCKET_SIZE 10   // default value is 10
```

I created a data structure `Task` to represent a media file to be processed. In addition, there is another data structure called `TaskManager` to manage all of the tasks to be done in buckets.

#### Task.h: **struct Task**

```C++
struct Task
{
  std::string file_dir;         /* Directory. */
  std::string file_name;        /* Filename. */
  int album_pk;                 /* Album pk. */
  enum Instruction instruction; /* Instruction to perform. */
  
  // ...other helper function
};
```

`TaskManager` stores these `struct Task`'s in `std::list`, with each instance of `std::list` represents a bucket. Then these buckets are contained in `std::vector` and the whole tasks data is define as `tasks` in `TaskManager`.

#### Task.h: **TaskManager::tasks**

```C++
class TaskManager : pubic QObject
{
  // ...
public:
  std::vector<std::list<struct Task> > tasks;
  // ...
}
```

The use of different containers results from the way the program retrieves these data in my application. In my design, the worker threads exporting the photos will be picking up buckets to process when a worker thread is available or has done processing its previous bucket. Therefore, I would need to be able to access a random item in these buckets, making `std::vector` a good fit. While within each bucket, media files are only processed sequentially so `std::list` would be a better fit. There are a lot of references about the differences between `std::vector` and `std::list`, here is [one](https://www.geeksforgeeks.org/difference-between-vector-and-list/).

Obviously, we don't want multiple worker threads processing the same bucket. Therefore, `TaskManager` maintains another shared state `buk_to_process` to indicate the index of the next bucket to process. As mentioned above, since multiple threads need to read and write to `buk_to_process`, we will need a synchronization tool to manage concurrent access and `TaskManager` keeps a mutex `buk_m` to do just that.

#### Task.h: **TaskManager::buk_to_process**

```C++
class TaskManager : pubic QObject
{
  // ...
public:
  size_t buk_to_process = 0;    /* Bucket index to be processed. */
  std::mutex buk_m{};           /* Synchronize BUK_TO_PROCESS. */
  // ...
}
```

### Spawn the worker threads

The thread library is part of the C++ std library and is defined in header [`<thread>`](https://en.cppreference.com/w/cpp/thread/thread/thread). To spawn a thread, call the constructor and pass in the function you would need the thread to execute and all the required arguments of that function. My implementation will first initialize an instance of `TaskManager`, collect all tasks in buckets and put these tasks in `task_manager`, an instance of `TaskManager`, and spawn the threads to process these task buckets. Also, the thread spawning all these worker threads is referred as the main thread, which is the only thread since the start of the program until we explicitly call `std::thread ()` to create other threads. The code to spawn the worker threads is in `MainWindow::act_on_next ()`. Note that we need to pass a reference of `task_manager` since we need all worker threads to access to the same instance of `TaskManager`.

#### mainwindow.cpp: **act_on_next**

```C++
void
MainWindow::act_on_next ()
{
  // ...collect tasks and put them in task_manager (instance of class TaskManager)

  /* Spawns threads to process the task. */
  std::thread thread_pool[WORKER_CNT];
  for (size_t worker = 0; worker < WORKER_CNT; worker++)
    {
      thread_pool[worker] = std::thread (TaskExe::process_bucket,
                                         std::ref (task_manager), job);
    }

  /* UPDATE_PROGRESS () blocks until all tasks are processed. */
  update_progress (task_manager);

  for (size_t worker = 0; worker < WORKER_CNT; worker++)
    thread_pool[worker].join ();
}
```

`TaskExe::process_bucket` defines the exporting execution. First each work thread is responsible to look up the index of the next bucket to process, pick it up, and update the index so that the next worker thread can work on the next bucket. This implementation is done in a static function `pick_up_bucket(TaskManager &)`. The worker thread now needs to lock `buk_m` before writes to `buk_to_process` to avoid a data race with other worker threads. Note that the recommended way of using a mutex is to wrap it inside a `lock_guard` so that the mutex is released (unlocked) automatically when the `lock_guard` object goes out of scope. This is known as the `RAII-style` and the cppreference shows an example [here](https://en.cppreference.com/w/cpp/thread/lock_guard).

#### TaskExe.cpp: **pick_up_bucket**

```C++
static const size_t
pick_up_bucket (TaskManager &task_manager)
{
  const size_t last_bucket_idx = task_manager.tasks.size () - 1;
  const std::lock_guard<std::mutex> lock (task_manager.buk_m);
  const size_t buk_idx = task_manager.buk_to_process;

  // TODO (refactor) raise custom exception instead of the standard one.
  if (buk_idx > last_bucket_idx)
    throw std::exception ();

  task_manager.buk_to_process++;

  return buk_idx;
}
```

### Use QProgressDialog to Indicate the Progress

While worker threads are exporting the photos, what is main thread supposed to do? At the very minimum, the main thread has to know whether the tasks have been completely processed. Otherwise, the user might just close the application while worker threads are still working. The simplest way to wait for all worker threads is to call [`std::thread::join`](https://en.cppreference.com/w/cpp/thread/thread/join). However, after the call the main thread will block until the called upon worker thread has finished executing. This will create very bad user experience since the application will appear to hang.

Therefore, I added a progress dialog in `TaskManager` so that the main thread can update the progress of the works being done. The user is still not able to interact with other part of the application while the progress dialog shows on the screen. But at least the user gets some indication that the app is not crashing or stalling. The code implementing the progress update is in `TaskManager::update_progress()`. You could follow the [documentation](https://doc.qt.io/qt-6/qprogressdialog.html) that shows how to use `QProgressDialog`.

#### Task.cpp: **update_progress**

```C++
void
TaskManager::update_progress ()
{
  auto cur_value = _pd->value ();
  for (auto i = 0; i < _processed_cnt; i++)
    {
      std::cout << "set value to " << i << std::endl;
      _pd->setValue (cur_value + i);
    }

  _pd->setValue (cur_value + _processed_cnt);
  _processed_cnt = 0;
  ready = false;
}
```

### Communication between main thread and worker threads

In updating the progress (i.e. `_pd->_setValue (cur_value + _processed_cnt)` in the above code block), the main thread has to know how much progress has been done and each worker thread will have its own progress to report. This creates another shared state (i.e. the progress of the entire exporting task) and needs additional handling of the synchronization. Here I used another synchronization tool `condition_variable`  to manage the concurrency issue.

`std::condition_variable::wait`  still requires a `mutex` (`cv_m` as defined in `TaskManager`) to manage the concurrent access to the shared state, but different from a simple `mutex`, you could additionally pass in a predicate (some function that will return a boolean). Here the predicate is whether there is any progress to be update by the main thread. In my implementation, after a worker thread has done processing one bucket, it will increment the progress in `_processed_cnt` in `TaskManager` and call `std::condition_variable::notify_one` so that a thread waiting for the same mutex could continue its execution. Note the waiting thread could be a worker thread trying to give an update of its own progress or the main thread trying to increment the progress in the progress dialog.

Implementation of the worker thread's progress updating is in `complete_task`.

#### TaskExe.cpp: **complete_task**

```C++
static void
complete_task (TaskManager &task_manager, const int success_cnt,
               const int error_cnt)
{
  const std::lock_guard<std::mutex> lock (task_manager.cv_m);
  auto processed_cnt = success_cnt + error_cnt;
  task_manager.decrement_tasks (processed_cnt);
  task_manager.cv.notify_one ();
}
```

Implementation of the main thread's progress updating is in `update_progress`.

#### mainwindow.cpp: **update_progress**

```C++
static void
update_progress (TaskManager &task_manager)
{
  size_t remaining_cnt;
  do
    {
      std::cout << "awaiting tasks to be processed..." << std::endl;
      std::unique_lock<std::mutex> cv_m (task_manager.cv_m);
      auto &ready = task_manager.ready;
      task_manager.cv.wait (cv_m, [&ready] { return ready; });
      task_manager.update_progress ();
      remaining_cnt = task_manager.remaining_cnt ();
    }
  while (remaining_cnt > 0);
}
```

The progress update by the main thread is wrapped in a do while loop since it's likely that main thread has to update the progress dialog in several iterations until all of the task buckets are exported. Note that the main thread exits the do while loop when the `remaining_cnt > 0` is `false`. This `remaining_cnt` is actually another shared state within `TaskManager` and is protected by the same mutex `cv_m` for concurrent access. If you trace through the code where `remaining_cnt` is accessed, you should see there is a mutex in place before the the shared state is read or written.

## Conclusion

A lot is going on under the hood to make multi-threading implementation work as intended. Bugs in data races are known to be hard to track and it is recommended to minimize the scope of the `critical section` established by the mutex. Since when we call lock on a mutex, any other thread trying to enter the critical section will have to wait until the mutex having acquired thread unlocks, essentially turning the concurrent execution into sequential execution and limiting the benefit of concurrency. However, mutex is the necessary tool to manage the shared state access. Programmers have to be aware how mutex is used in their code, what shared state the mutex is trying to protect and the scope of the critical section the mutex establishes. The C++ standard library also provides a higher level library [`<future>`](https://en.cppreference.com/w/cpp/header/future) for concurrency. When the use case is applicable, it is actually recommended to use the higher level library `future` than manage the threads in your own implementation. Definitely check it out when you're trying incorporate concurrency into your own project.
