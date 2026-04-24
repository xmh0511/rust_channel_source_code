Looking at this implementation of multiple-producer single-consumer, which was the implementation of `channel` in Rust's standard library; however, Rust's memory order model is derived from C++. So, it should be based on the C++ standard to do the **formal reasoning**.

The code of the `Queue` structure is
````rust
impl<T> Queue<T> {

    pub fn new() -> Queue<T> {
        let stub = unsafe { Node::new(None) };
        Queue { head: AtomicPtr::new(stub), tail: UnsafeCell::new(stub) }
    }

    /// Pushes a new value onto this queue.
    pub fn push(&self, t: T) {
        unsafe {
            let n = Node::new(Some(t));
            let prev = self.head.swap(n, Ordering::AcqRel);
            (*prev).next.store(n, Ordering::Release);
        }
    }

    pub fn pop(&self) -> PopResult<T> {
        unsafe {
            let tail = *self.tail.get();
            let next = (*tail).next.load(Ordering::Acquire);

            if !next.is_null() {
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
```` 
And the major codes of `shared` are
````rust
impl<T> Packet<T> {
  pub fn send(&self, t: T) -> Result<(), T> {
        // See Port::drop for what's going on
        if self.port_dropped.load(Ordering::SeqCst) {
            return Err(t);
        }

        if self.cnt.load(Ordering::SeqCst) < DISCONNECTED + FUDGE {
            return Err(t);
        }

        self.queue.push(t);
        match self.cnt.fetch_add(1, Ordering::SeqCst) {
            -1 => {
                self.take_to_wake().signal();
            }
           n if n < DISCONNECTED + FUDGE =>{/*...*/}
            // Can't make any assumptions about this case like in the SPSC case.
            _ => {}
        }

        Ok(())
    }
      pub fn recv(&self, deadline: Option<Instant>) -> Result<T, Failure> {
        // This code is essentially the exact same as that found in the stream
        // case (see stream.rs)
        match self.try_recv() {
            Err(Empty) => {}
            data => return data,
        }

        let (wait_token, signal_token) = blocking::tokens();
        if self.decrement(signal_token) == Installed {
            if let Some(deadline) = deadline {
                let timed_out = !wait_token.wait_max_until(deadline);
                if timed_out {
                    self.abort_selection(false);
                }
            } else {
                wait_token.wait();
            }
        }

        match self.try_recv() {
            data @ Ok(..) => unsafe {
                *self.steals.get() -= 1;
                data
            },
            data => data,
        }
    }
    fn decrement(&self, token: SignalToken) -> StartResult {
        unsafe {
            assert_eq!(
                self.to_wake.load(Ordering::SeqCst),
                EMPTY,
                "This is a known bug in the Rust standard library. See https://github.com/rust-lang/rust/issues/39364"
            );
            let ptr = token.to_raw();
            self.to_wake.store(ptr, Ordering::SeqCst);

            let steals = ptr::replace(self.steals.get(), 0);

            match self.cnt.fetch_sub(1 + steals, Ordering::SeqCst) {
                DISCONNECTED => {
                    self.cnt.store(DISCONNECTED, Ordering::SeqCst);
                }
                // If we factor in our steals and notice that the channel has no
                // data, we successfully sleep
                n => {
                    assert!(n >= 0);
                    if n - steals <= 0 {
                        return Installed;
                    }
                }
            }

            self.to_wake.store(EMPTY, Ordering::SeqCst);
            drop(SignalToken::from_raw(ptr));
            Abort
        }
    }

    pub fn try_recv(&self) -> Result<T, Failure> {
        let ret = match self.queue.pop() {
            mpsc::Data(t) => Some(t),
            mpsc::Empty => None,
            mpsc::Inconsistent => {
                let data;
                loop {
                    thread::yield_now();
                    match self.queue.pop() {
                        mpsc::Data(t) => {
                            data = t;
                            break;
                        }
                        mpsc::Empty => panic!("inconsistent => empty"),
                        mpsc::Inconsistent => {}
                    }
                }
                Some(data)
            }
        };
        match ret {
            // See the discussion in the stream implementation for why we
            // might decrement steals.
            Some(data) => unsafe {
                if *self.steals.get() > MAX_STEALS {
                    match self.cnt.swap(0, Ordering::SeqCst) {
                        DISCONNECTED => {
                            self.cnt.store(DISCONNECTED, Ordering::SeqCst);
                        }
                        n => {
                            let m = cmp::min(n, *self.steals.get());
                            *self.steals.get() -= m;
                            self.bump(n - m);
                        }
                    }
                    assert!(*self.steals.get() >= 0);
                }
                *self.steals.get() += 1;
                Ok(data)
            },

            // See the discussion in the stream implementation for why we try
            // again.
            None => {
                match self.cnt.load(Ordering::SeqCst) {
                    n if n != DISCONNECTED => Err(Empty),
                    _ => {
                        match self.queue.pop() {
                            mpsc::Data(t) => Ok(t),
                            mpsc::Empty => Err(Disconnected),
                            // with no senders, an inconsistency is impossible.
                            mpsc::Inconsistent => unreachable!(),
                        }
                    }
                }
            }
        }
    }
}
````
PS: The complete codes are in [shared](https://github.com/rust-lang/rust/blob/34115d040b43d9a0dcc313c6282520a86d1e6f61/library/std/src/sync/mpsc/shared.rs).

To focus on the major logic, I commented out some code that is irrelevant to discuss `recv` and `send` here. 

Assuming three threads calling `send` to push the data. According to [[intro.races] p14](https://eel.is/c++draft/intro.races#14)
> If a side effect X on an atomic object M happens before a value computation B of M, then the evaluation B takes its value from X or from a side effect Y that follows X in the modification order of M.

If there are no other extra synchronizations, the `pop` thread is not guaranteed to see the subsequent nodes set in the `push` threads because no happen-before forces the visibility; that is, to see these modifications after the initial side effect in the mod order. However, the initial value `head` and its field `next` to be `null` is guaranteed to be visible. 

As per [[atomic.order] p10](https://eel.is/c++draft/atomics.order#10)
> Atomic read-modify-write operations shall always read the last value (in the modification order) written before the write associated with the read-modify-write operation.

If one RMW thereof reads the initial value `0` of `cnt`, other RMW operations must read the later modification in the modification order, and no two RMW operations can read the same modification. All operations on `cnt` are RMW operations; this guarantees that the load part of the RMW operation can always make forward progress and can't step on another(i.e., all operations are serialized and cannot overlap). With the synchronization established by the serialized RMW operations(i.e., `cnt.fetch_xxx`) on `cnt`, the loads to `next` and `head` are guaranteed to see the later values in their modification orders, because there are happen-before relationships between side effects to `next` or `head` and the loads.

Each `swap` in `push` can establish synchronization with another, however, since `fetch_add(1,...)` is **sequenced-after** the call of `push`, there is no happen-before established by synchronization in `push` for any two `fetch_add`s, hence, no constraint on these `fetch_add(1,...)`s for their order in the modification order; there can be a case where the `fetch_add` corresponding to the first node other than `head` can be ordered to later in its modification order. Specifically, I tried to illustrate this case in the following image:

<img width="946" height="769" alt="pzSXoXYf" src="https://github.com/user-attachments/assets/7e68efa6-ecaf-4caa-867d-890969c1e4a7" />



The first `pop` starts from the `head` and with `head->next` is `null`, the `cnt.fetch_sub(1)` synchronizes-with `M1`, this can only guarantee that `node2->next = node3` happens before `try_recv()` that is sequenced after the `cnt.fetch_sub(1)`. If I didn't reason wrong in my picture, `head->next = node1` is still unordered to `head->next.load(...)` in `try_recv`(i.e., in `queue.pop()`), however, `self.head.load` is guaranteed to see `node3` because of the synchronization established by RMW operations on `cnt`. Therefore, the `try_recv` should step into the `Inconsistent` branch to spin a loop and wait for the update, even though this still cannot guarantee seeing the new value, as per [atomics].order] p12
> The implementation should make atomic stores visible to atomic loads, and atomic loads should observe atomic stores, within a reasonable amount of time.

That is, the loop can technically be an infinite loop because this requirement is not forced by the standard(even though it's not possible in a real implementation). 

So, in this case, even using the `cnt`, the channel still can run into the `InConsistent` status and enter the loop to wait for the node value to be visible. Not only in this case, if we swap the order of `M1` and `cnt.fetch_sub(1)` in the picture, we can still end up in an inconsistent state. Specifically, `cnt` will be `-1` after `cnt.fetch_sub(1)`, and the thread that does `recev` will be parked. Then `M1` reads `-1` and tries to `unpark` the parked thread. Even though the `unpark` operation synchronizes with the parked thread, the visibility is the same as in the above analyzed case.
