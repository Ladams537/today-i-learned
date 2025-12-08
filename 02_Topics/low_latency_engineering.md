## Memory Pool

```rust
// A "slot" in our pool can be one of two things
enum Slot<T> {
    Occupied(T),
    Free(Option<usize>), // Points to the index of the next free slot
}

pub struct MemoryPool<T> {
    // The contiguous block of memory
    pool: Vec<Slot<T>>, 
    
    // The index of the first available slot (The "Head")
    // We use Option because the pool might be full (None)
    first_free: Option<usize>, 
}
```

### Minimal Overhead
There is one catch. In C++, a `union` of an `int` and a `double` takes up 8 bytes (the size of the largest member). In Safe Rust, an enum needs to know which one it is, so it adds a "tag" (usually 1 byte + padding). So `slot<f64>` might take 16 bytes instead of 8.

```rust
impl<T> MemoryPool<T> {
    // We need to pass in the data ('item') we want to store!
    // We return 'usize' (the index) so the caller knows where we put it (like a handle).
    fn allocate(&mut self, item: T) -> usize {
        
        match self.first_free {
            Some(index) => {
                // CASE 1: RECYCLE
                // We have an empty spot at 'index'. 
                
                // 1. Check the slot to find the *next* free index
                // We use 'if let' because we know this slot MUST be Free 
                if let Slot::Free(next_free) = self.pool[index] {
                    // 2. Update our head pointer to the next spot
                    self.first_free = Some(next_free);
                } else {
                    panic!("Corrupt memory pool: first_free pointed to occupied slot!");
                }

                // 3. Overwrite the slot with the new data
                self.pool[index] = Slot::Occupied(item);
                
                // Return the index (the "handle")
                index
            }
            
            None => {
                // CASE 2: EXPAND
                // No gaps in the middle. We must append to the end.
                self.pool.push(Slot::Occupied(item));
                
                // The index is simply the last spot in the vector
                self.pool.len() - 1
            }
        }
    }
}
```

```rust
fn deallocate(&mut self, index: usize) {
    // 1. Create the note. The note should point to whatever first_free holds right now.
    // If first_free was Some(3), our new note says Some(3).
    // If first_free was None (full), our new note says None.
    let note_for_next_person = self.first_free;

    // 2. Put the note inside the locker (mark it as Free)
    self.pool[index] = Slot::Free(note_for_next_person);

    // 3. Update the Head to point to this locker
    self.first_free = Some(index);
}
```


## Implementing Generational Safeguards
```rust
// The secure badge we give to the user
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Handle {
    pub index: usize,
    pub generation: u64,
}

// The internal container for every spot in the pool
struct Entry<T> {
    generation: u64,     // Persists forever, increments on free
    payload: Slot<T>,    // The actual data or the free pointer
}

pub struct GenerationalPool<T> {
    entries: Vec<Entry<T>>,
    first_free: Option<usize>,
}
```

```rust
impl<T> GenerationalPool<T> {
    pub fn get(&self, handle: Handle) -> Option<&T> {
        // 1. Bounds check
        if handle.index >= self.entries.len() {
            return None;
        }

        let entry = &self.entries[handle.index];

        // 2. THE GENERATION CHECK (The "Expiration Date")
        // The handle says "I am looking for Gen 1". 
        // If the entry says "I am currently Gen 2", the handle is stale.
        if handle.generation != entry.generation {
            return None;
        }

        // 3. THE OCCUPANCY CHECK
        // Even if generation matches, we make sure it's actually data, not a Free pointer.
        match &entry.payload {
            Slot::Occupied(data) => Some(data),
            Slot::Free(_) => None, // Should be impossible if generations match, but good for safety
        }
    }
}
```

```rust
fn deallocate(&mut self, handle: Handle) -> Result<(), &'static str> {
    // 1. Safety Check: Is the handle pointing to a valid index?
    if handle.index >= self.entries.len() {
        return Err("Index out of bounds");
    }

    // 2. Security Check: Is the handle expired?
    // If the entry is Gen 2, but the handle is Gen 1, we reject it.
    if handle.generation != self.entries[handle.index].generation {
        return Err("Handle is stale (Generation mismatch)");
    }

    // 3. The "Invalidation" Step
    // We increment the generation. Now any other copy of 'handle' in the wild is useless.
    self.entries[handle.index].generation += 1;

    // 4. The "Free List" Logic (Standard Link-up)
    // Point this slot to the OLD head
    let old_head = self.first_free;
    self.entries[handle.index].payload = Slot::Free(old_head);
    
    // Update head to point to this slot
    self.first_free = Some(handle.index);

    Ok(())
}
```

## Spin Locking
* **Standard Mutex**: When a thread tries to lock it and fails, it goes to sleep. The OS wakes it up later. This saves battery/CPU, but the "context switch" (going to sleep and waking up) takes time (microseconds)
* **Spin Lock**: When a thread tries to lock and fails, it enters a `while` loop, effectively shouting "Are we there yet? Are we there yet?" millions of times per second

```rust
pub fn lock(&self) {
    loop {
        // 1. The "Test-and-Set": Try to grab the lock once.
        if self.locked.compare_exchange(
            false, 
            true, 
            Ordering::Acquire, 
            Ordering::Relaxed
        ).is_ok() {
            return; // We got it!
        }

        // 2. The "Test": Wait while it looks locked (Read-Only Loop)
        // This keeps the cache line in "Shared" state, avoiding bus traffic.
        while self.locked.load(Ordering::Relaxed) == true {
            // Tell the CPU we are spinning (saves power/optimizes pipeline)
            std::hint::spin_loop(); 
        }
    }
}
```

The importance of doing read only operations while you are waiting for the lock to free. Writes are extremely expensive. Similar problem that many modern databases face -> trade-off between read/write/security/etc

## Rate Limiter
Ring Buffer comes back here again. The industry standard.
**Concept**
Imagine a clock face with 60 ticks (seconds). Instead of storing every request, we just have 60 buckets.
* Bucket 0: Count of requests that happened at second 0
* Bucket 1: Count of requests that happened at second 1
**Gives us**
* O(1) Updates: Just increment an integer in an array
* Fixed Memory: It never grows, no matter how much traffic we get
* **The Logging**: If you want to keep that data for analytics ("How many users did we block yesterday?"), that is a separate concern. You would emit an event to a logging system asynchronously, but you never keep that history inside the hot rate-limiter memory. That would violate the "Fixed Memory" rule

### Sliding Window
* `T` seconds
* `N` small buckets
* Request comes:
    * What bucket are we in?
    * If this bucket belongs to the current second? Discard if not
    * Add 1 to the bucket
    * Sum all buckets. If sum > K, block request

```rust
use std::sync::Mutex;
use std::time::{Duration, Instant};

struct RateLimiterState {
    buckets: Vec<usize>,
    bucket_times: Vec<u64>,
    start_time: Instant,
}

pub struct RateLimiter {
    state: Mutex<RateLimiterState>,
    window_size_secs: usize,
    max_requests: usize,
}

impl RateLimiter {
    pub fn new(window_size_secs: usize, max_requests: usize) -> Self {
        Self {
            state: Mutex::new(RateLimiterState {
                buckets: vec![0; window_size_secs],
                bucket_times: vec![0; window_size_secs], // Initialise with 0
                start_time: Instant::now(),
            }),
            window_size_secs,
            max_requests,
        }
    }

    pub fn check_and_update(&self) -> bool {
        let mut state = self.state.lock().unwrap();
        
        // 1. Get current time "tick"
        let current_tick = state.start_time.elapsed().as_secs();
        let index = (current_tick as usize) % self.window_size_secs;

        // 2. Lazy Reset: If this bucket belongs to an old second, clear it
        if state.bucket_times[index] != current_tick {
            state.buckets[index] = 0;
            state.bucket_times[index] = current_tick;
        }

        // 3. Sum total requests in the window
        // (Optimisation: We could cache the 'total' and update it incrementally, 
        // but summing a small vector is fast enough for now).
        let total_requests: usize = state.buckets.iter().sum();

        if total_requests < self.max_requests {
            // 4. Update: Allow request
            state.buckets[index] += 1;
            true
        } else {
            // Block request
            false
        }
    }
}
```

## SPSC Queue
Great analogy for the `ordering` segment:
* Good kitchen counter (buffer),
* plate of food (data -> item),
* service bell (head index) analogy
* if service bell is dinged early (because it is easier than making the plate of food) -> the consumer (tail) will be met with an empty plate

Thus `.store()` is not just assigning a number, but publishing the state of the "world" to other threads.

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::cell::UnsafeCell;
use std::mem::MaybeUninit;

// A fixed-size Lock-Free Queue
pub struct SpscQueue<T, const N: usize> {
    // The buffer holds the data. 
    // We use UnsafeCell because we are sharing it between threads without a Mutex.
    // We use MaybeUninit because we don't want to initialize N items on creation.
    buffer: UnsafeCell<[MaybeUninit<T>; N]>,

    // ---------------------------------------------------------
    // THE MECHANICAL SYMPATHY PART
    // ---------------------------------------------------------

    // Producer writes to this.
    // We align it to 64 bytes so it starts on its own cache line.
    #[repr(align(64))] 
    head: AtomicUsize,

    // Consumer writes to this.
    // We align it to 64 bytes so it starts on its own DIFFERENT cache line.
    // This physically forces 'tail' to be at least 64 bytes away from 'head'.
    #[repr(align(64))]
    tail: AtomicUsize,
}

// We must manually tell Rust: "It is safe to share this across threads"
unsafe impl<T, const N: usize> Sync for SpscQueue<T, N> {}

impl<T, const N: usize> SpscQueue<T, N> {
    pub fn new() -> Self {
        // Assertion to ensure N is a power of 2 (makes math faster)
        assert!(N > 0 && (N & (N - 1)) == 0, "Size must be power of 2");
        Self {
            buffer: UnsafeCell::new(unsafe { MaybeUninit::uninit().assume_init() }),
            head: AtomicUsize::new(0),
            tail: AtomicUsize::new(0),
        }
    }

    pub fn push(&self, item: T) -> Result<(), T> {
        // 1. Load indices
        // Relaxed load is fine for head because only WE (producer) write to it.
        let head = self.head.load(Ordering::Relaxed);
        // Acquire load for tail ensures we see the consumer's latest reads.
        let tail = self.tail.load(Ordering::Acquire);

        // 2. Check if full
        if head.wrapping_sub(tail) >= N {
            return Err(item); // Queue is full
        }

        // 3. Write data
        // Calculate slot: head % N (Optimized to bitwise AND if N is power of 2)
        // & is a cool mathematical optimisation
        let slot = head & (N - 1);
        unsafe {
            // Get raw pointer to the slot and write the data
            let ptr = self.buffer.get() as *mut MaybeUninit<T>;
            (*ptr.add(slot)).write(item);
        }

        // 4. Commit (Publish)
        // Release ordering tells CPU: "Make sure the data write (step 3) happens 
        // BEFORE anyone sees this incremented index."
        self.head.store(head.wrapping_add(1), Ordering::Release);
        Ok(())
    }

    pub fn pop(&self) -> Option<T> {
        // 1. Load indices
        let tail = self.tail.load(Ordering::Relaxed);
        let head = self.head.load(Ordering::Acquire);

        // 2. Check if empty
        if tail == head {
            return None;
        }

        // 3. Read data
        let slot = tail & (N - 1);
        let item = unsafe {
            let ptr = self.buffer.get() as *const MaybeUninit<T>;
            (*ptr.add(slot)).assume_init_read()
        };

        // 4. Commit
        self.tail.store(tail.wrapping_add(1), Ordering::Release);
        Some(item)
    }
}
```
