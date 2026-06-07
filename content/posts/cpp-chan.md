---
title: "Porting panama to C++: concurrency patterns from a channel"
date: 2026-06-07
draft: false
tags: ["c++", "concurrency", "rust", "channels", "systems"]
description: "Porting Lambels/chanus (Rust) to C++20: mutex + condvar, the wait-and-signal pattern, spurious wakeups, and graceful close protocol."
cover:
  image: "/images/cpp-chan-cover.png"
  alt: "Flappy Bird-style green pipe sprite"
  caption: "Pipe sprite from OpenGameArt (CC0, public domain)"
  relative: false
---

I found the crust of rust series from Jon Gjengset to be a priceless tool for learning low level programming paired with concurrency primitives.
Through my study of the guide I wanted to port a lot of the things written there to C++. This was in an effort to better understand C++ but also to
better appreciate the footguns that Rust natively avoids using the borrow checker.

I think that in order to appreciate Rust and better understand its principles it is good to start with a more permissive language like C or C++. So lets
explore this implementation.

> Watch the crust of rust series [here](https://www.youtube.com/watch?v=rAl-9HwD858&list=PLqbS7AVVErFiWDOAVrPt7aYmnuuOLYvOa)
>
> Find the full code [here](https://github.com/Lambels/cpp-chan) (a ⭐ is greatly appreciated)
> I really suggest reading [this](https://github.com/Lambels/cpp-chan/blob/main/include/chan.hpp) file before continuing with the article, the code is short and concise. 

## The shape of the problem
Really as the first step in this implementation we need to identify what we are building.

> [!IMPORTANT]
> Unbounded MPSC Channel.

This instantly reveals some implementation details:
1. MP - Multi producer channels (the producer side must be copyable meaningfully)
2. SC - Single consumer side (the consumer can't be copied, can only be moved)
3. Unbounded - this means that the channel buffer grows without limit hence producers never block.

These restrictions set the tone for one of the simplest cross thread boundary channel implementations.

At the center of our implementation will be a shared `Inner` type. This type will provide the shared state across the two ends of the channel.
Already from this description it is kind of clear how the `Inner` type should be stored but we will get to that later. Lets sketch out the `Inner` struct.

```cpp
namespace chan::detail {

template <typename T>
struct Inner {
    std::queue<T> queue;                // the actual queue of messages
    std::mutex mu;                      // the mutex guarding all these fields
    std::condition_variable available;  // condition variable to allow waiting on wakeup condition
    size_t senders = 1;                 // number of senders 
    bool closed = false;                // closed flag
};

}  // namespace chan::detail
```

## Mutex + condvar: the wait-and-signal pattern
This is the very important concurrency pattern that we explore with this implementation. It allows us to serialise two states of a thread:
- sleeping and waiting on a signal
- doing work

To implement this we need two concurrency primitives: `std::condition_variable` and `std::mutex`.

### What a mutex can't express on its own
A mutex is a tool which allows for mutually exclusive access to data. We call regions which access the protected data: **critical regions**. The mutex 
put simply ensures that sequences of these critical regions are serialised across thread boundaries: they don't overlap.

This is the safest condition we can apply, if we are in a critical region we know we are the only one in the critical region 
and hence we can safely modify any shared data. 

Although the mutex does allow blocking behaviour it isn't what we would want here, we would want the receiver side of the channel to block until 
there is a message to be consumed. If we would only implement this using a mutex the code would look something like this:
```
while true
    lock(mutex)
    check condition
    release(mutex)
    sleep(1ms)
```
This of course is completely sub optimal as it is very CPU intensive, a spin loop to continuously check a condition hogs up CPU time. Another reason 
why this implementation is horrible is because it creates a lot of contention on the lock, we can see that as a sender, it would be very hard to
acquire the lock to send a message, we would only have the `1ms` time frame to acquire the lock.

What we really need is to describe a pattern which allows the receiver to give up the lock and wait for a signal to wake up. This would ideally be
an OS operation which signal putting the thread to sleep and waking it up only when the signal is generated, this pattern is exactly why the 
condition variable was invented.

### The atomic unlock-wait-wake-relock dance
The implementation of `std::condition_variable` is a very light weight one. I will try to describe from a high level.
The `std::condition_variable` has an internal wait queue guarded by a small local spinlock to guarantee mutual exclusion. The api provides three main 
methods:
1. `wait(lock, predicate)`
2. `notify_one()`
3. `notify_all()`

The `wait` method parks the thread on the internal queue and atomically gives up the lock.
> [!IMPORTANT]
> These two operations happen atomically as per to avoid the lost signal bug.
> If we were to do these two operations in two separate steps: free the mutex and then park the thread 
> a signal from `notify_*` methods could sneak in between mutex free and parking of the thread hence losing the signal 
> the `condition_variable` is the tool above the `mutex` which allows for this atomic switch between the two states.
> 
> This is why it is important that the `mutex` you give up in the wait is the same `mutex` you acquire before checking the condition and 
> sending a notification, otherwise the whole contract is broken and signals can be lost.

```cpp
std::unique_lock<std::mutex> lock(inner_->mu); // acquire the lock
// checking some predicate ...

// predicate is undesirable, we need to wait 
inner_->available.wait(lock, [this] { // atomically give up the lock and park the thread
    return !inner_->queue.empty() || inner_->senders == 0;
});
```

Here we can see the pattern in action, first we acquire the lock on the mutex, we use the `std::unique_lock` guard because the `condition_variable`
needs to free the lock and re acquire the lock, the unique_lock allows for these operations (the basic `std::lock_guard` is just scoped, unlocks on drop).

After we check the predicate for the desired outcome, it is crucial that anyone who modifies the output of this predicate does so through the same mutex 
we acquired, then if the predicate value isn't satisfactory we go to sleep while atomically releasing the lock, this transfers the control to any 
sender who would possibly want add a message to the queue.

## The predicate covers two wakeup reasons
Now looking closer to the wait call, we covered the first argument (unique lock) now lets examine the predicate. This is the actual predicate / condition 
from which the name `condition_variable` comes from, we only want to wake up if this is true. 

In our case its pretty understandable that, as a receiver, we only want to be awake if there is a message on the queue or if there are no more senders 
left (this means we have to exit). 

### Spurious wakeups, and why the predicate-as-loop pattern handles them for free
The above call to wait expands to this under the hood, this is a very powerful way to use `wait`.

```cpp
// what wait(lock, pred) effectively does:
// we have the lock acquired here.
while (!predicate()) {
    cv.wait(lock); // we give it up here
}
```

We basically want to keep on waiting until the predicate is true (if the predicate was already true notice that we don't even wait we just return).
But why would we need to loop? The sender in my case only notifies the receiver after the predicate is already true... The reason is that spurious 
wake-ups can occur. The implementation of cond vars doesn't guarantee that waking up happened from a signal, you could wake from a signal or from the 
OS randomly deciding to wake up your thread (there are precise reasons for this, I will not cover them). Point being, that you only want to really 
wake up if you have any updates on the predicate so this says: if I wake up and the predicate is still false (spurious wakeup or bad signal) just go
back to sleep.

So the whole `wait(lock, pred)` line is blocking until predicate is true, you are certain that after that line the predicate is true and that you 
have a lock acquired, that's it!

## Closing the channel from both sides (gracefully...)
There are 2 main concerns when it comes to proper closing:
1. Is the memory freed? 
2. Do subsequent or inflight calls to `.recv()` and `.send()` gracefully signal the closing of the channel or do they hang?

### The memory
We made use of the `std::shared_ptr` smart pointer. This allows for the main `Inner` object to have a shared ownership between 
the senders and receiver. As it was not clear who should own this object.
- Senders: can be multiple, which one owns it?
- Receiver: can technically own the object but the implementations (I tried them) lead to re creating a shared object state, invariably 
there needs to be shared state for the graceful shutdown, we put that in `Inner` and wrap it with `std::shared_ptr` to keep it around
for both the senders and receivers.

This way when all the participants in the channel's sides get dropped we free the shared state.

### Senders close when the count hits zero

```cpp
~Sender() {
    if (!inner_) return;  // moved
    bool last;
    {
        std::lock_guard<std::mutex> lock(inner_->mu);
        inner_->senders--;
        last = (inner_->senders == 0);
    }
    if (last) inner_->available.notify_one();
}
```

When dropping senders we keep track of the current available senders (we do this under mutex guard since this is shared state). If we are 
the last sender then we signal to the waiting receiver (if waiting) that there is no more data coming, this allows for the receiver to gracefully
exit and not hang forever. We use the condition variable for this because observing the predicate from before:

```cpp
return !inner_->queue.empty() || inner_->senders == 0;
```

It is one of the conditions we wait for on the other side. 

### Receivers close by setting the flag
On the receiver side, when we exit we acquire the lock before dropping and set the closed flag to true:

```cpp
~Receiver() {
    if (!inner_) return;
    std::lock_guard<std::mutex> lock(inner_->mu);
    inner_->closed = true;
}
```

This way any sender: 

```cpp
bool send(T val) {
    {
        std::lock_guard<std::mutex> lock(inner_->mu);
        if (inner_->closed) return false;
        inner_->queue.push(std::move(val));
    }
    inner_->available.notify_one();
    return true;
}
```

first checks if the receiver is still on the other side. If it isn't, we return false to signal that we didn't send anything through the channel.

> [!NOTE]
> Notice that we always check for `!inner_` this is because the `Receiver` allows for move operations hence we could be dropping a moved
> `Receiver` which would have a nulled out `inner_`.

### Why `send` checks `closed` under the lock
It is very possible that dropping the sender and sending a message could happen at the same time, this would lead to a race condition between 
senders reading the close flag and the receiver setting the closed flag. This is UB and could lead to `send` returning `true` even if there is no one 
on the other side to receive the message.

## Some considerations
This implementation is not one to one with Gjengset's original implementation and I don't claim it to be the same, I just try to port the ideas 
from Rust to C++ and explain them.

There are some tests (minimal) written in the github repo and also an iterator implementation for the receiver, but I found them out of scope for this
blog in which I aimed to cover the concurrency patterns.

A lot of things could have been handled differently, we could have used `shared_ptr` and `weak_ptr` for the `Receiver` and `Sender` ends of the channel, allowed
for `Sender` and `Receiver` generation from one another and more. This implementation is by no way good...

## Up next: bounded MPSC and lock-free SPSC
We currently tackled the simplest channel there is, there are a lot of different flavours and implementations to channels:
- lock free
- MPMC
- SPSC
- sync/async

I will make sure to come back with more implementations along the way.
