# Tokio Overview

## A First Look at Tokio

Tokio is a Rust asynchronous runtime library. The underlying implementation is based on cross-platform multiplexing IO and event loops like epoll/kqueue. It is currently supporting [io_uring](https://github.com/tokio-rs/tokio-uring). Its scheduler is similar to Erlang/Go's implementation of N:M threads, where threads execute a Task to take advantage of multiple cores. Task is a green thread based on the Future abstraction of Rust, which is ideal for IO-intensive applications because it does not require pre-allocation of extra stack memory and can create a large number of tasks.

Although Rust itself does not provide an asynchronous runtime, as mentioned in [1.1](./01_intro_async.md), we can easily use third-party runtimes such as Tokio because of Rust's powerful macros.

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;  // listen

    loop {
        let (mut socket, _) = listener.accept().await?; // async wait for incoming tcp socket

        tokio::spawn(async move {                       // create async task and let Tokio process it
            let mut buf = vec![0; 1024];

            loop {                                      // read and write data back until EOF
                let n = socket.read(&mut buf).await?;   // async wait for incoming data

                if n == 0 { return; }

                socket.write_all(&buf[0..n]).await?;    // async wait socket is ready to write and write data
            }
        });
    }
}
```

As seen in the above code, it is easy to write a common TCP server in Tokio, where the main thread listens on a port and accepts connections in a loop, and each connection is processed in a Future, which gives up CPU to other futures when waiting for IO, so we have a high performance, high concurrency TCP server. This code will also be used as an example for subsequent code interpretations.

## Architecture Overview

The "magic" here is in the `#[tokio::main]` macro, which preprocesses the code to look like this:

```rust
fn main() {
    tokio::runtime::Builder::new_multi_thread().enable_all()
        .build().unwrap()
        .block_on(async {
            // async main
        })
}
```

After the program starts, various data and IO resources are initialized with the build call, the worker thread is started and then the `async` code block is run in the main thread, which is the `async main` we wrote.

![tokio overview](./assets/01_tokio_overview.png)
[link](https://excalidraw.com/#json=5711793527717888,Dc5N5AvwyvKQ7jcngGlmLg)

The above diagram depicts the general architecture of the Tokio runtime using echo as an example, which is useful for understanding Tokio and will be mentioned again later.

The worker threads on the right side of the graph are generally the same as the number of cores, and execute the futures submitted by `tokio::spawn`. When there is no executable task, it will poll events via epoll/kqueue, which is the responsibility of the reactor. When awakened by events, it will continue to try to execute tasks and so on.

On the left side of the figure, the runtime calls poll on our main function future passed to `block_on` on the main thread, and returns `Pending` when executing `listener.accept().await?`, so the main thread will be suspended (park), that means it waits for a semaphore and hibernates. Three events follow: 

* (wake1) When a TCP connection is received, the worker thread will get events in `poll events` and send a semaphore to the main thread. The main thread is woken up from `park` and then executes `tokio::spawn`. The worker thread continues the loop, i.e. poll events.

* (wake2) The main thread in `tokio::spawn` will first put the future of the TCP connection into the run queue, then wake up the worker thread and go back to waiting for TCP accept. The worker thread will take the task from the run queue and execute it after it is woken up, that is `let mut buf = vec![0; 1024];` the beginning of the code.

  When the worker thread executes `socket.read(&mut buf).await?`, it cannot read because the data is not ready yet and returns `Pending`, then executes other tasks or waits for IO events.

* (wake3) When the OS receives TCP data, the worker thread receives events and puts the previously unexecuted task into the run queue, then takes it out of the run queue and executes it, where it calls syscall `read` to read the received data and finally writes the data back to the client. Then if the client closes the connection, the task is finished and the worker thread will execute other tasks or wait for events.

There are two things worth noting here. First, there are multiple `tasks run queues`, including each worker's own queue and the global queue, and workers will prefer to fetch tasks from their own queue, as explained in detail in [3.2](./03_task_scheduler.md). The other is that multiple worker threads will execute tasks concurrently, but only one worker thread will act as a reactor to poll events, and the new events may be executed by the reactor itself or by other worker threads for the corresponding future.

## Code directory and structure

[`Cargo.toml`](https://github.com/tokio-rs/tokio/blob/master/Cargo.toml) shows that Tokio is a workspace containing several sub-crates, mainly including `tokio`, `tokio-macros`, `tokio-stream`, `tokio-util` and other code such as tests and examples. `tokio-stream` is the implementation of [Stream](https://docs.rs/futures-core/0.3.14/futures_core/stream/trait.Stream.html) and `tokio-util` is for Tokio users. :et's ignore them for now. The main code is in the `tokio` and `tokio-macros` sub-crates. Let's take a look at their code size:

```rust
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
# tokio/src
Rust                           256           6332          23616          26233

# tokio-macros/src
Rust                             3             50            266            410
```

The code is mainly concentrated in `tokio`, and there are about the same number of comments, so there is a lot of documentation and comments written. There are more than 20,000 lines of code in total, which is not a lot. There is also a lot of test code, and some code is used to implement some asynchronous versions of the standard library, so we actually don't need to pay much attention to the code at the beginning.

In addition, there are some major dependencies:

- [bytes](https://github.com/tokio-rs/bytes): Tools for handling bytes.
- [mio](https://github.com/tokio-rs/mio): Encapsulated cross-platform IO operations, such as epoll, kqueue, etc.
- [parking_lot](https://github.com/Amanieu/parking_lot): Implements many synchronization primitives, such as locks and semaphores.

Let's take a look at the submodules of tokio:

```bash
.
# core
├── lib.rs        // library file
├── blocking.rs   // Provides encapsulation of blocking operations
├── coop.rs       // Helper to achieve better collaborative scheduling
├── future        // Some encapsulation of future operations
├── park          // Similar to `std::thread::park`, but more generic
├── runtime       // The core of the Tokio runtime, including event loop, task management, scheduling, thread pooling, etc.
├── sync          // Tools for different tasks to synchronize, such as channel and Mutex
├── task          // The task described above

# async std in Tokio
├── io            // Wrapper for IO operations, equivalent to asynchronous std::io, and the basis for building submodules such as net, fs, etc.
├── net           // TCP/UDP/Unix wrapper, similar to std::net
├── fs            // Asynchronous std::fs
├── process       // Asynchronous process management, such as the ability to run a child process asynchronously, similar to std::process
├── signal        // Asynchronous signal processing, such as ctrl-c
├── time          // Time-related modules, such as Sleep

# utils
├── loom          // Unifies the interface between std and loom(github.com/tokio-rs/loom) to facilitate testing
├── macros        // Some public macros are mainly declaration macros. And tokio-macro is mainly a procedural macro 
└── util          // Common util module for tokio internal code 
```

At first we focus on the modules in the core section, the io and net modules involved in the echo example, and the macros and util modules involved in the code.

In addition, Tokio uses some [feature flag](https://docs.rs/tokio/1.5.0/tokio/index.html#feature-flags) to allow customization of some features, such as `rt-multi-thread`, which turns on the multi-thread scheduler, and `full`, which turns on almost all features and in pracitve, you can turn off some of as needed.