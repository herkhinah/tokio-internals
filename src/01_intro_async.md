# Introduction to Rust async

Before we talk about Tokio, we have to talk about asynchronous programming in Rust, because it is different from many languages.

If you are familiar with Rust Async, you can skip this chapter.

For languages like Erlang/Go/Nodejs, asynchronous runtime is built into the language itself, right out of the box. But Rust, as a system-level language, doesn't want to be limited to one implementation, so it goes the other way and provides basic features like Future/async/await, implementing a task similar to Nodejs Promise, but leaving the scheduling and running future to third-party implementations like Tokio](https://github.com/tokio-rs/tokio), [async-std](https://github.com/async-rs/async-std), etc. For example, in the figure below, on the left is the basic functionality provided by the Rust language that drives the execution of the task, while the right is the scheduling mechanism that the third-party runtime needs to implement.

![](./assets/01_rust_overview.png)
[https://excalidraw.com/#json=5287000177377280,_EjA-elJg02sgC71T8uXLQ](https://excalidraw.com/#json=5287000177377280,_EjA-elJg02sgC71T8uXLQ)


## Future, async, await

`Future` is a Rust trait (similar to interface) that represents an asynchronous task, a zero-cost abstraction, a lightweight thread (similar to promise) that is scheduled and executed by the runtime. The `Future` trait needs to implement the `poll` method, determines if the task is finished, and returns `Ready` if it is finished (e.g. IO data ready), otherwise it returns `Pending`.

Our code does not call poll directly, but executes the future with Rust's keyword `.await`. `await` will be compiled by Rust to generate code to call `poll`, and if it returns `Pending`, it will be hung by runtime (e.g., put back in the task queue). When an event is generated, the hung future will be woken up and Rust will call `poll` on the future again, and if it returns `Ready`, execution is complete.

In addition to implementing the Future trait directly, a function or a block of code can be turned into a Future via an `async`, where the `.await` of another future can be called to wait for the child future to become `Ready`.

```rust
struct HelloFuture { ready: bool, waker: ... }

impl Future for HelloFuture {          // 1. custom Future(leaf)
    fn poll(self: Self, ctx: &mut Context<'_>) -> Poll<()> {
        if self.ready {
            Poll::Ready(())
        } else {
            // store waker in ctx somewhere
            Poll::Pending
        }
    }
}

async fn hello_world() {                 // 2. generated Future by async
    println!("before await");
    HelloFuture { ready: false }.await;  // 3. HelloFuture is pending, then park
    println!("Hello, world!");
}

fn main() {
    let task = hello_world(); // 4. task is a generated Future
    // ... reactor code is ignored here, which will wake futures
    runtime::spawn(task);     // 5. task is run by runtime
}
```

For example, in the above code, `HelloFuture` is a `Future` implemented by implementing the `Future` trait, `hello_world` is a Future turned into an `async` function and is passed into the runtime for execution (here `runtime::spawn` is just an example). `hello_world` calls `HelloFuture.await` again, because ready is false, so `hello_world` will be hung until HelloFuture is awakened.

The above nested futures can form a Future tree, and generally the leaf nodes are implemented by the runtime (e.g. Tokio) itself by implementing Future traits, such as io, tcp, time, etc. The non-leaf nodes are implemented by the library code. The non-leaf nodes are implemented by the library code or by the user through `async` calls to different child futures. root future is submitted to the runtime for execution, runtime calls root future through the scheduler, and then root calls poll down the hierarchy.

![](./assets/01_future_tree.png)
[https://excalidraw.com/#json=6697978081312768,u9VYjoGonibMqcPX8VWeGg](https://excalidraw.com/#json=6697978081312768,u9VYjoGonibMqcPX8VWeGg)

## 生成状态机和 stackless

Future 会被 Rust 编译为一个状态机的代码，当执行到子 Future 的 `await` 的时候，会进入下一个状态，所以下次执行时可以从 `await` 的地方继续执行。因此 Rust 不需要预先为 future 分配独立的栈（stackless），是 zero-cost abstraction。但也因为如此，future 只能在 `await` 的地方调度走，是 cooperation scheduling（协同调度），而且很难做抢占式调度，这点和 stackful 的 Go/Erlang 不一样。状态机的示意伪代码如下：

## Generating stackless state machines

Future is compiled by Rust as code into a state machine, and when the execution reaches the `await` of a child Future, it goes to the next state, so the next execution can continue from the `await`. So Rust does not need to pre-allocate a separate stack for future (stackless), it is a zero-cost abstraction. However, because of this, a future can only be scheduled at `await`, which is cooperative scheduling, and it is difficult to do preemptive scheduling, which is different from stackful Go/Erlang. The schematic pseudo-code for the state machine is as follows.

```rust
use std::{future::Future, task::Poll};

enum HelloWorldState {
    Start,
    Await1(HelloFuture),
    Done,
}

impl Future for HelloWorldState {
    type Output = ();
    fn poll(&mut self: HelloWorldState, ctx: &mut Context<'_>) -> Poll<Output> {
        match self {
            HelloWorldState::Start => {
                println!("before await"); // code before await

                let hello = HelloFuture { ready: false };

                *self = HelloWorldState::Await1(hello);
                self.poll();              // re-poll after first state change
            },
            HelloWorldState::Await1(hello) => {
                match hello.poll(ctx) {      // await by poll
                    Poll::Pending => {
                        Poll::Pending
                    },
                    Poll::Ready(output) => {
                        println!("Hello, world!"); // code after await
                        *self = HelloWorldState::Done;
                        let output = ();
                        Poll::Ready(output)
                    }
                }
            },
            HelloWorldState::Done => {
                panic!("can't go here")
            }
        }
    }
}
```

In this schematic doe, `async fn hello_world()` is turned into a state machine with an `enum` and its poll method. The initial state is `Start` and the first time `poll` is executed the code before `.await` is executed and the state changes to `Await1`. The next time it is polled, since the state is `Await1`, it will enter the second branch and execute `hello.poll()`, returning `Pending` if `hello` is not finished, otherwise it will execute the code after `.await`.

As you can see, Rust mainly provides these basic tools and code generation, while how to manage OS threads, how to schedule tasks, how to poll events, how to wake up pending tasks, etc., all need to be implemented by runtime itself, which can be implemented as an N:M model similar to Erlang/Go, or as a single-threaded event-driven model like Nodejs.

Usually the runtime will encapsulate some common modules, such as IO/TCP/timer, etc. Just be careful to use the runtime encapsulated ones instead of the standard library ones when writing code. A runtime like Tokio also provides a dedicated thread pool (similar to Erlang's dirty scheduler) for running such blocking operations, thus not affecting the execution of reactor or other tasks.

### Reference

* [https://rust-lang.github.io/async-book/](https://rust-lang.github.io/async-book/)
* [https://www.infoq.com/presentations/rust-async-await/](https://www.infoq.com/presentations/rust-async-await/)
