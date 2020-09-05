---
title: Non-Blocking Queues
layout: post
---

Shared memories/variables are widely used in parallel programs. To ensure the correctness, concurrent access must be synchronized. It is commonly known using locks to guarantee exclusive access, yet locks are not free.

<!--more-->

  - if a thread T is preempted while holding a lock, no thread that needs the lock can progress until T runs again and releases the lock.
  - if a low priority thread holds a lock is interrupted by an interrupt or signal handler that needs the lock, deadlock can occur since the lock is still held by the thread

Using non-blocking data structures solves these problems. Non-blocking data structures are based on the idea that: every reachable concrete state of the data structure corresponds to a unique abstract state, and can serve as the starting point for any operation (or the continuation of any operation already underway). There is no reachable state in which a thread has to wait for action on the part of some other thread in order to make progress. And we will soon see the non-blocking queue implementation where the clean up state could be performed by any threads.

### CAS 

To implement the non-blocking data structures, we take advantage of the hardware atomic primitive instructions. Being atomic means that a task either is completely done or is not done at all; there is no state like half done or half not done. These instructions include of course the loads and stores. 

There are also instructions to read, modify and rewrite memory locations as a single hardware atomic operation. Such instructions have different names depending on the CPU architectures. For example, on X86, Sparc, there is `bool compare_and_swap(a, e, n)` or `CAS` which will replace the word at address `a` with the value `n`, provided that it contained `e` before.

Instruction `CAS` will return `true` if update is successful (value on address is not changed) and `false` otherwise. The typical idiom for `CAS` is:

```java
do {
  e = *a
  n = f(e)
} while (!CAS(a, e, n))
```
Note that a while loop is used so that if the update fails the code will continue trying until it succeeds. 

The `CAS` alternatives on MIPS, ARM, RISC V, etc, are decomposed further to 2 atomic primitive instructions: `val load_linked(a)` which returns word at address `a` and tags the cache line as a side effect; `bool store_conditional(a, n)` which writes n to a if tagged line is still present. 

And the typical idiom for `LL` and `SC` is: 


```java
do {
 e = LL(a)
 n = f(e)
} while (!SC(a, n))
```


These two mechanisms (i.e. `CAS` vs `LL,SC`) have pros/cons, yet it is beyond the scope of this article. And we will continue to focus on `CAS`.


But `CAS` is not a panacea. One of the well known issues is the ABA problem with CAS. The ABA problem happens in case that the memory location value was A, then is changed to B, then changed back to a bitwise equal but semantically different value A. For`CAS` the value is still `A` and it assumes that nothing has changed at all. ABA problem does not bother LL/SC (using cache tag). We can use counted pointers to avoid ABA problems, i.e. `<serial_no, addr>` pair, and increase serial number on every change.



### Michael-Scott Queue 

The Michael-Scott queue is one of the commonly applied non-blocking data structures. It is first mentioned  in the paper `Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms` published in 1996 by M. Michael and M. Scott. In this paper, the authors presented a non-blocking queue implementation with `CAS` which we are going to have a take a look at soon.

In Java, the `CAS` operation is provided by `AtomicReference#compareAndSet`. We define our node and queue class as following: 

```java

class Queue<E> {

  class Node<E> {
    E item;
    AtomicReference<Node<E>> next;
    Node(E item, AtomicReference<Node<E>> next) {
      this.item = item;
      this.next = next;
    } 
  }
  
  Node<E> dummy = new Node<>(null, null);
  AtomicReference<Node<E>> head = new AtomicReference<>(dummy);
  AtomicReference<Node<E>> tail = new AtomicReference<>(dummy);
}
```

##### Enqueue

As we know, a queue is defined by two operations, i.e. `enqueue` and `dequeue`. For the non-blocking queue, they are a little more complicated. The `enqueue` is now a two-step operation:

1. CAS next pointer of tail node to new node
2. Use CAS to swing the tail pointer


In the algorithm `enqueue`, we first create a new node `newNode`. In the loop, we read the tail node `currentTail` and the successor of `currentTails` which is named as `tailNext`.

During the process reading the tail node and the it's successor, there's possibility that some other threads change either or even both of them (finished either 1st step or all two steps of the enqueue process). And we need to consider all the possibilities to make sure the algorithm is correct. 

1. If some other thread T has already finished enqueue, the `currentTail` becomes obsolete and is no longer equal to `tail.get()`, i.e. `currentTail != tail.get()` (tail pointer has been swang to some new node appended by thread T). In this case, we need to return to the beginning of the loop and reread the tail node. 
2. If some other thread T has just finished the first step of enqueue, i.e. append a new node to the tail, but not yet updates the tail pointer (`currentTail == tail.get() && tailNext != null` ). The tail pointer points to some node whose successor is non trivial, it means we need to do some clean up (to finish step 2) and any thread can do the clean up on behalf of any thread.
3. If the `tailNext` is null and `currentTail` equals to `tail.get()`, we can try to add the `newNode` to the queue tail by `currentTail.next.compareAndSet(null, newNode)`. If this `CAS` fails, we restart the loop. And if it succeeds, we can continue to swing the tail pointer `tail.compareAndSet(currentTail, newNode)`, and note that 2hen we CAS the tail pointer to the new node, and if it fails, we just don't care, because it means some other thread has already done it on our behalf!

```java
public void enqueue(E item) {
  Node<E> newNode = new Node<>(item, null);
  while (true) { // keep trying
    Node<E> currentTail = tail.get();
    Node<E> tailNext = currentTail.next.get(); // if no other threads change the tail, it should be null 
  // snapshot taken, now 
  if (currentTail == tail.get()) { // no body changed the tail pointer yet, so it can continue
      if (tailNext != null) { // oops, someone has already added a node to the end
        tail.compareAndSet(currentTail, tailNext) //OK, I do the clean-up,try to update tail pointer 
      } else {
        if (currentTail.next.compareAndSet(null, newNode)) { // add new node to tail
          tail.compareAndSet(currentTail, newNode) // try to update tail pointer, if it fails? we don't care
          return;
        }
      }
    }
  }
}
```

##### Dequeue

For the dequeue operation, if we look at the head and tail and if they are different, we can safely dequeue, otherwise we have to discriminate between the cases: the queue is empty `next == null`; some other thread just finished step 1 of enqueue, and we will do the clean up on its behalf `tail.compareAndSet(last, next)`.

Generally, the dequeue operation can be summarized as the following 4 steps: 

1. Read data in dummy's next node
2. CAS head pointer to dummy's next node
3. Discard old dummy node
4. Node from which we read is the new dummy node


Note that the item value has to be read before CAS, because once the CAS is done, any other thread is allowed to do a dequeue which will remove the node which we want to read value from. That's the reason why we have `E item = next.item` before `head.compareAndSet(first, next)`. 


```java
public E dequeue() {
  while (true) {
    // take a snapshot
    Node<E> first = head.get();
    Node<E> last = tail.get();
    Node<E> next = first.next.get();
    
    if (first == head.get()) { // nobody changed the head yet 
      if (first == last) { // empty or some other thread executed step 1 of enqueue
        if (next == null) return null; // empty
        tail.compareAndSet(last, next); // we do the clean up 
      } else {
        E item = next.item; 
        if (head.compareAndSet(first,next)) {
          return item;
        }
      }
    }
  }
}
```


### ConcurrentLinkedQueue

As the old saying goes, "Don't reinvent wheels," Java offers us the `ConcurrentLinkedQueue` that, I quote from the Java Doc: 

> The implementation employs an efficient "wait-free" algorithm based on one described in Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms by Maged M. Michael and Michael L. Scott

If there is any difficulty in understanding the `CAS`, just use the `ConcurrentLinkedQueue`, enjoy and have fun!


