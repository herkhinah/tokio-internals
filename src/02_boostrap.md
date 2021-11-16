# Tokio runtime start

This is the code for the [echo example](https://github.com/tokio-rs/tokio/blob/df10b68d47631c3053342c7cf0b1fab8786565b2/examples/echo.rs):

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

Unlike normal synchronous code, Tokio requires us to write an async main function, which relies on the `#[tokio::main]` macro to generate the code, which is explained in detail in the [documentation](https://docs.rs/tokio/1.5.0/tokio/attr.main.html), so we won't go over it here. In fact, we can also call the API in our main function to complete the runtime initialization as needed, instead of going through Tokio's default macro.

```rust
fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
            let listener = TcpListener::bind("127.0.0.1:8080").await?;
            // ...
        })
}
```

## Runtime initialization

This is the `build()` function (simplified and future code examples will be simplified as appropiate):

```rust
let (driver, resources) = driver::Driver::new(self.get_cfg())?;

let (scheduler, launch) = ThreadPool::new(core_threads, Parker::new(driver));
let spawner = Spawner::ThreadPool(scheduler.spawner().clone());

// Create the blocking pool
let blocking_pool = blocking::create_blocking_pool(self, self.max_blocking_threads + core_threads);
let blocking_spawner = blocking_pool.spawner().clone();

// Create the runtime handle
let handle = Handle {
    spawner,
    io_handle: resources.io_handle,
    blocking_spawner,
};

// Spawn the thread pool workers
let _enter = crate::runtime::context::enter(handle.clone());
launch.launch();

Ok(Runtime {
    kind: Kind::ThreadPool(scheduler),
    handle,
    blocking_pool,
})
```
[link](https://github.com/tokio-rs/tokio/blob/a5ee2f0d3d78daa01e2c6c12d22b82474dc5c32a/tokio/src/runtime/builder.rs#L540)

Several resources are initialized:

- `runtime::driver::Driver` is the driver of the event loop, for example IO events. The `poll events` and `events` from [1.2](./01_intro_tokio.md) are implemented using this.
- `ThreadPool` is the worker from [1.2](./01_intro_tokio.md). Since poll events are performed in the worker, the driver needs to be passed in. When the worker has no task to execute it will be "parked" by the `Parker`. `park` is actually a call to the driver to wait for events.
- `blocking_poll` is a pool of threads dedicated to running blocking tasks, where `core_threads` are hteads to run workers (because earch worker itself can be considered as an blocking task), and the rest are dedicated to running blocking tasks.
- runtime `handle` and `runtime`, including the driver and thread pool, are returned to main at the end. In the following chapters running lightweight threads will be called worker threads and running other blocking tasks will be called blocking threads. 

### IO driver initialization

Let's focus on the driver constructor `driver::Driver::new`:

```rust
let poll = mio::Poll::new()?;
let waker = mio::Waker::new(poll.registry(), TOKEN_WAKEUP)?;
let registry = poll.registry().try_clone()?;

let slab = Slab::new();
let allocator = slab.allocator();

let io_driver = Driver {
    tick: 0,
    events: Some(mio::Events::with_capacity(1024)),
    poll,
    resources: Some(slab),
    inner: Arc::new(Inner {
        resources: Mutex::new(None),
        registry,
        io_dispatch: allocator,
        waker,
    }),
};

return Resources {
    io_handle: Handle {
        inner: Arc::downgrade(&io_driver.inner),
    },
    ...
};
```
[link](https://github.com/tokio-rs/tokio/blob/a5ee2f0d3d78daa01e2c6c12d22b82474dc5c32a/tokio/src/io/driver/mod.rs#L114)

Serveral resources are initialized here:

- the `poll` object from mio is at it's core an epoll/kqueue syscall wrapper.
- `waker` is created by registering a special event `TOKEN_WAKEUP` with  `poll`, which is used to wake up the worker thread directly (wake2 in figure [1.2](./01_intro_tokio.md)) and execute the task after being woken up.
- `Slab` is similar to [Slab](https://en.wikipedia.org/wiki/Slab_allocation) in the Linux kernel, which allocates memory for objects and returns addresses. Currently only the IO driver uses Slab to allocate `ScheduledIo` to keep the status of IO resources and waker related information. Slab is used because it reduces memory fragmentation and improves utilization whe are large number of IO events are generated and cleared. More on this in [3.1](./03_slab_token_readiness.md).
- `io_driver` contains the previously created resources and its handle is shared by threads to access some data, such as poll, Slab, etc.

### Thread pool initialization

```rust
// ThreadPool::new:
cores.push(Box::new(Core {
    run_queue,
    ...
    park: Some(park),
}));

remotes.push(Remote {
    steal,
    ...
    unpark,
});

let shared = Arc::new(Shared {
    remotes: remotes.into_boxed_slice(),
    inject: queue::Inject::new(),
    ...
});

launch.0.push(Arc::new(Worker {
    shared: shared.clone(),
    index,
    core: AtomicCell::new(Some(core)),
}));
```
[link](https://github.com/tokio-rs/tokio/blob/a5ee2f0d3d78daa01e2c6c12d22b82474dc5c32a/tokio/src/runtime/thread_pool/mod.rs#L46)

There is not much to say about worker initialization, just pay attention to a few structs.

- `Core` - represents a worker's own data, such as run_queue, park and is allocated on the heap via [`Box`](https://doc.rust-lang.org/std/boxed/index.html).
- `Remote` - `steal` is a copy of the `run_queue` in `Core` used to let other threads "steal" tasks.
- `Shared` - shared by all workers, holds `Remote` and the global `run_queue`, etc. It is eventually returned as the left tuple value `scheduler`.
- `Worker` - Contains `shared` and `core` from previously and gets put into the right tuple return value `launch`.


The blocking_pool similarly initializes a run queue and some state, but it's a bit simpler.

### Starting the Thread pool

`let _enter = crate::runtime::context::enter(handle.clone());` will assign the handle to the [thread local](https://doc.rust-lang.org/std/thread/struct.LocalKey.html) key [`CONTEXT`](https://github.com/tokio-rs/tokio/blob/a5ee2f0d3d78daa01e2c6c12d22b82474dc5c32a/tokio/src/runtime/context.rs#L7). `Context` will be restored to its previous value when `_enter` is dropped before the runtime builder returns. The reason for this "detour" is that `runtime::spawn_blocking` called from `launch.launch()` needs to get the runtime from the current thread local. After that all worker threads are started (`launch.launch()`) with the following code:

```rust
pub(crate) fn launch(mut self) {
    for worker in self.0.drain(..) {
        runtime::spawn_blocking(move || run(worker));
    }
}
```
[link](https://github.com/tokio-rs/tokio/blob/a5ee2f0d3d78daa01e2c6c12d22b82474dc5c32a/tokio/src/runtime/thread_pool/worker.rs#L277)

When `runtime::spawn_blocking` is called, the anonymous function `|| run(worker)` is passed in, which is the actual logic that the worker thread will execute.

As follows, the anonymous function is wrapped in a `BlockingTask` and placed in the blocking thread's run queue so that when it runs it will execute the anonymous function. Since there are not enough threads at this point, a new OS thread is initialized (if there are idle thread, they are notified via a condvar) and the blocking thread logic is started. Each worker occopies a blocking thread and runs in the blocking thread until the end.

```rust
// runtime::spawn_blocking:
let (task, _handle) = task::joinable(BlockingTask::new(func));

let mut shared = self.inner.shared.lock();
shared.queue.push_back(task);

let mut builder = thread::Builder::new(); // Create OS thread
// run worker thread
builder.spawn(move || {
    rt.blocking_spawner.inner.run(id);
})
```
[link](https://github.com/tokio-rs/tokio/blob/a5ee2f0d3d78daa01e2c6c12d22b82474dc5c32a/tokio/src/runtime/handle.rs#L201)

## Summary

The `runtime::Runtime` is now constructed and contains almost all the data for the Tokio runtime to run, such as the io driver, thread pool, etc. The worker thread is also created and running, but let's put the worker thread aside for now and look at the subsequent execution of the main thread in the next chapter, which is the `block_on` method of the Runtime in the second code paragraph of this chapter.