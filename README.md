## Experiment 1.2: Understanding how it works.

After calling `spawner.spawn(...)`, the program immediately prints the message `"(Added) Spawned the timer task and immediately continued execution."`. This shows that spawning an asynchronous task does not block the main thread. The async task runs independently and begins executing its own code, printing `"Rahardi's Komputer: howdy!"`. After that, it awaits `TimerFuture::new(...)`, which introduces a 2-second delay using a custom future. Once the timer completes, the task resumes and prints `"Rahardi's Komputer: done!"`. This behavior demonstrates how Rust's async runtime schedules and executes tasks in a non-blocking manner, allowing the main thread to continue while background tasks wait or perform work asynchronously.

![Timer output screenshot](asset\timer.png)

## Experiment 1.3: Multiple Spawn and Removing Drop

Spawning creates and schedules a new asynchronous task to be executed by the executor. When we call `spawner.spawn(...)`, the task is put into a queue managed by the executor. This allows us to run multiple tasks concurrently without blocking the main thread. In this experiment, we spawn three tasks that each print a message, wait for 2 seconds using a timer, and then print a completion message. The effect of spawning is that all three tasks run independently and are managed by the executor in an asynchronous, non-blocking fashion.

- The `Spawner` is responsible for submitting new tasks into the executor’s ready queue. It holds the sending end of a channel (`SyncSender`) that communicates with the `Executor`. Whenever we call `spawn(...)`, it packages the future into a `Task` and sends it into the executor’s queue for execution. Without the spawner, we wouldn't be able to initiate or schedule any asynchronous work.

- The `Executor` receives tasks from the queue and continuously polls them to drive their progress. It is the core runtime that handles task execution. The executor runs in a loop, pulling each task from the queue and polling its future until completion. It ensures that tasks get a chance to run, pause, and resume based on readiness and availability of resources like timers.

- Calling `drop(spawner)` tells the executor that no more tasks will be submitted. Since the executor waits for tasks using a blocking `recv()` call, it will wait forever unless the sending side of the channel (i.e. the `Spawner`) is dropped. Dropping the spawner closes the channel, and `recv()` will then return `Err`, allowing the executor to gracefully exit its loop once all tasks are finished. If we do **not** drop the spawner, the program hangs indefinitely after completing all tasks because the executor still expects more incoming work.

* The `Spawner` is the interface used to schedule work.
* The `Executor` processes the scheduled work.
* `drop(spawner)` is the signal that says: "All tasks have been scheduled. You can shut down when you're done."
  Together, they form a minimal async runtime system. Without dropping the spawner, the executor will never finish, even if all tasks are done. Without the executor, tasks are never run. Without the spawner, no tasks are ever scheduled. These three parts are tightly coupled to make async execution work correctly.

**Screenshot 1: With `drop(spawner)`**
![with drop](asset\with_drop.png)

**Screenshot 2: Without `drop(spawner)`**
![no drop](asset\no_drop.png)

This experiment demonstrates that spawning allows multiple tasks to be scheduled concurrently. The spawner and executor must work together to execute these tasks. Dropping the spawner is essential to allow the executor to shut down properly. Without it, the program will hang, waiting for tasks that will never come. This experiment clarifies the relationship between components in a minimal async runtime and shows the importance of cleaning up properly.