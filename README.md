# rust_channel_source_code

````rust
//! A mostly lock-free multi-producer, single consumer queue.
//!
//! This module contains an implementation of a concurrent MPSC queue. This
//! queue can be used to share data between threads, and is also used as the
//! building block of channels in rust.
//!
//! Note that the current implementation of this queue has a caveat of the `pop`
//! method, and see the method for more information about it. Due to this
//! caveat, this queue might not be appropriate for all use-cases.

// https://www.1024cores.net/home/lock-free-algorithms
//                          /queues/non-intrusive-mpsc-node-based-queue

#[cfg(all(test, not(target_os = "emscripten")))]
mod tests;

pub use self::PopResult::*;

use core::cell::UnsafeCell;
use core::ptr;

use crate::boxed::Box;
use crate::sync::atomic::{AtomicPtr, Ordering};

/// A result of the `pop` function.
pub enum PopResult<T> {
    /// Some data has been popped
    Data(T),
    /// The queue is empty
    Empty,
    /// The queue is in an inconsistent state. Popping data should succeed, but
    /// some pushers have yet to make enough progress in order allow a pop to
    /// succeed. It is recommended that a pop() occur "in the near future" in
    /// order to see if the sender has made progress or not
    Inconsistent,
}

struct Node<T> {
    next: AtomicPtr<Node<T>>,
    value: Option<T>,
}

/// The multi-producer single-consumer structure. This is not cloneable, but it
/// may be safely shared so long as it is guaranteed that there is only one
/// popper at a time (many pushers are allowed).
pub struct Queue<T> {
    head: AtomicPtr<Node<T>>,
    tail: UnsafeCell<*mut Node<T>>,
}

unsafe impl<T: Send> Send for Queue<T> {}
unsafe impl<T: Send> Sync for Queue<T> {}

impl<T> Node<T> {
    unsafe fn new(v: Option<T>) -> *mut Node<T> {
        Box::into_raw(box Node { next: AtomicPtr::new(ptr::null_mut()), value: v })
    }
}

impl<T> Queue<T> {
    /// Creates a new queue that is safe to share among multiple producers and
    /// one consumer.
    pub fn new() -> Queue<T> {
        let stub = unsafe { Node::new(None) };
        Queue { head: AtomicPtr::new(stub), tail: UnsafeCell::new(stub) }
    }

    /// Pushes a new value onto this queue.
    pub fn push(&self, t: T) {
        unsafe {
            let n = Node::new(Some(t));  // #1
            let prev = self.head.swap(n, Ordering::AcqRel);  // #2
            (*prev).next.store(n, Ordering::Release);  // #3
        }
    }

    /// Pops some data from this queue.
    ///
    /// Note that the current implementation means that this function cannot
    /// return `Option<T>`. It is possible for this queue to be in an
    /// inconsistent state where many pushes have succeeded and completely
    /// finished, but pops cannot return `Some(t)`. This inconsistent state
    /// happens when a pusher is pre-empted at an inopportune moment.
    ///
    /// This inconsistent state means that this queue does indeed have data, but
    /// it does not currently have access to it at this time.
    pub fn pop(&self) -> PopResult<T> {
        unsafe {
            let tail = *self.tail.get();
            let next = (*tail).next.load(Ordering::Acquire);  // #4

            if !next.is_null() { // #5
                *self.tail.get() = next;
                assert!((*tail).value.is_none());
                assert!((*next).value.is_some());
                let ret = (*next).value.take().unwrap();  
                let _: Box<Node<T>> = Box::from_raw(tail);
                return Data(ret);
            }

            if self.head.load(Ordering::Acquire) == tail { Empty } else { Inconsistent }
        }
    }
}

impl<T> Drop for Queue<T> {
    fn drop(&mut self) {
        unsafe {
            let mut cur = *self.tail.get();
            while !cur.is_null() {
                let next = (*cur).next.load(Ordering::Relaxed);
                let _: Box<Node<T>> = Box::from_raw(cur);
                cur = next;
            }
        }
    }
}

````

参考链接 https://edp.fortanix.com/docs/api/src/std/sync/mpsc/mpsc_queue.rs.html

`#3`和`#4`是对同一个原子对象的release和acquire操作从而形成synchronization, 使得`#1` *happen-before* `#5`避免data race。而`#2`出的`Ordering::AcqRel`是为了让前一个`push`中的由`swap`产生的写和后一个`push`中由`swap`产的的读(其读取结果是`prev`)产生synchronization, 因为后一个`push`中对`pre`的写操作需要看到前一个`push`中`Node::new`,因为`pre`指向前一个`push`中`Node::new`的对象，因此需要前一个`push`中的`Node::new` *happens-before* 后一个`push`中对该memory location(即`pre`)的写，从而避免data race。

`#2`是一个*read-modify-write*操作，这保证了一个线程中通过`swap`的读(`pre`)一定是另一个线程中`swap`写(`n`)的数据, 因此从`self.head`的modifiation order上看是这样的:
```
self.head: H
self.head: N1, H.next -> N1 
self.head: N2, N1.next -> N2
self.head: N3, N2.next -> N3
self.head: N4, N3.next -> N4
.
.
.
self.head: Nn, N(n-1).next -> Nn
```



Details can be seen [cnt.md](https://github.com/xmh0511/rust_channel_source_code/blob/main/cnt.md)
