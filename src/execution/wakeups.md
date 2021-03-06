# Task Wakeups with `LocalWaker` and `Waker`

It's common that futures aren't able to complete the first time they are
`poll`ed. When this happens, the future needs to ensure that it is polled
again once it is ready to make more progress. This is done with the
`LocalWaker` and `Waker` types.

Each time a future is polled, it is polled as part of a "task". Tasks are
the top-level futures that have been submitted to an executor.

`LocalWaker` and `Waker` each provide a `wake()` method that can be used to
tell the executor that their associated task should be awoken. When `wake()` is
called, the executor knows that the task associated with the `Waker` is ready to
make progress, and its future should be polled again.

`LocalWaker` and `Waker` also implement `clone()` so that
they can be copied around and stored. The difference between the two is
thread-safety: `LocalWaker` is `!Send` and `!Sync`, and so cannot be used from
threads other than the one it was created from. This allows `LocalWaker`
implementations to perform special optimized behavior for the current thread.
`Waker`s, on the other hand, are `Send` and `Sync`, and so can be used across
multiple threads. A `LocalWaker` can be turned into a thread-safe `Waker` using
the `into_waker()` function. This function is free to call-- it doesn't
allocate at runtime or anything similar, but calling `wake()` on the resulting
`Waker` may be less performant than calling `wake()` on the original
`LocalWaker`.

Let's try implementing a simple timer future using `Waker` and `LocalWaker`.

## Applied: Build a Timer

For the sake of the example, we'll just spin up a new thread when the timer
is created, sleep for the required time, and then signal the timer future
when the time window has elapsed.

Here are the imports we'll need to get started:

```rust
#![feature(arbitrary_self_types, futures_api, pin)]

use std::{
    future::Future,
    pin::{Pin, Unpin},
    sync::{Arc, Mutex},
    task::{LocalWaker, Poll, Waker},
    thread,
    time::Duration,
};
```

Let's start by defining the future type itself. Our future needs a way for the
thread to communicate that the timer has elapsed and the future should complete.
We'll use a shared `Arc<Mutex<..>>` value to communicate between the thread and
the future.

```rust
struct TimerFuture {
    shared_state: Arc<Mutex<SharedState>>,
}

/// Shared state between the future and the thread
struct SharedState {
    /// Whether or not the sleep time has elapsed
    completed: bool,

    /// The waker for the task that `TimerFuture` is running on.
    /// The thread can use this after setting `completed = true` to tell
    /// `TimerFuture`'s task to wake up, see that `completed = true`, and
    /// move forward.
    waker: Option<Waker>,
}

// Pinning will be covered later-- for now, it's enough to understand that our
// `TimerFuture` type doesn't require it, so it is `Unpin`.
impl Unpin for TimerFuture {}
```

Now, let's actually write the `Future` implementation!

```rust
impl Future for TimerFuture {
    type Output = ();
    fn poll(self: Pin<&mut Self>, lw: &LocalWaker)
        -> Poll<Self::Output>
    {
        // Look at the shared state to see if the timer has already completed.
        let mut shared_state = self.shared_state.lock().unwrap();
        if shared_state.completed {
            Poll::Ready(())
        } else {
            // Set waker so that the thread can wake up the current task
            // when the timer has completed, ensuring that the future is polled
            // again and sees that `completed = true`.
            shared_state.waker = Some(lw.clone().into_waker());
            Poll::Pending
        }
    }
}
```

Pretty simple, right? If the thread has set `shared_state.completed = true`,
we're done! Otherwise, we clone the `LocalWaker` for the current task,
convert it into a `Waker`, and pass it to `shared_state.waker` so that the
thread can wake the task back up.

Importantly, we have to update the `Waker` every time the future is polled
because the future may have moved to a different task with a different
`Waker`. This will happen when futures are passed around between tasks after
being polled.

Finally, we need the API to actually construct the timer and start the thread:

```rust
impl TimerFuture {
    /// Create a new `TimerFuture` which will complete after the provided
    /// timeout.
    pub fn new(duration: Duration) -> Self {
        let shared_state = Arc::new(Mutex::new(SharedState {
            completed: false,
            waker: None,
        }));

        // Spawn the new thread
        let thread_shared_state = shared_state.clone();
        thread::spawn(move || {
            thread::sleep(duration);
            let mut shared_state = thread_shared_state.lock().unwrap();
            // Signal that the timer has completed and wake up the last
            // task on which the future was polled, if one exists.
            shared_state.completed = true;
            if let Some(waker) = &shared_state.waker {
                waker.wake();
            }
        });

        TimerFuture { shared_state }
    }
}
```

Woot! That's all we need to build a simple timer future. Now, if only we had
an executor to run the future on...
