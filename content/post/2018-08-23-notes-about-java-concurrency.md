---
title: "Notes about java concurrency"
categories: ["tech"]
date: 2018-08-23
tags: ["java"]
---

# Thread safety
## Atomicity
If two threads share a variable, and the operations on that variable is not atomic, then problem might occur.
```java
public class TestThread {
    private long count = 0;
    public void inc() {
        count++;
    }
}
```
The operation `++` consists of three steps: read variable count, increase its value and write it back. So if there're two threads, the three steps might be interleaved, resulting in different final values of count.
### Solution: Compound actions
To address this problem, we need to make `++` an atomic operation, that is from the perspective of the other thread, `++` is either excecuted or not executed, can't be in half way. Java provides the data type called `AtomicLong`, its `incrementAndGet()` method is an atomic operation which will increase the value and return the increased value.

## Lock
We made `++` thread safe by using the `AtomicLong`, but if we try to share another variable and operate on both, problem will still occur.
```java
public class TestThread {
    private long count1 = new AtomicLong(0);
    private long count2 = new AtomicLong(0);
    public void inc() {
        count1.incrementAndGet();
        count2.incrementAndGet();
    }
}
```
Even both operation is atomic, the combination is not. To make the combination atomic, we need use lock.

Java provides `synchronized` to lock a section of code. Synchronized served as a mutex, meaning only one thread will be able to access the locked block a time. The syncronized is reentrant, so a lock can be reacquired by same thread. 

So a simple rework of above code becomes:
```java
public class TestThread {
    private AtomicLong count1 = new AtomicLong(0);
    private AtomicLong count2 = new AtomicLong(0);
    public synchronized void inc() {
        count1.incrementAndGet();
        count2.incrementAndGet();
    }
}
```

## Guard state with lock
For each mutable statee variable that may be accessed by more than one thread, all access should must be performed with the same lock held. This is called variable is guarded by lock.

Not only writing operation should be performed with lock held, but all the reading operations should hold the lock too.

If a invariant involes more than one variable, then all the variables involved in that invarint should be guarded by the same lock.

# Sharing variable
`Synchronized` not only prevents other thread from modifying the state when one thread is using it, but also ensures if one thread made a modification to the state, the change can become visible to other threads.

### Visibility
Consider the following example:
```java
public class NoVisibility {
    private static boolean ready;
    private static int number;
    private static class ReaderThread extends Thread {
        public void run() {
            while(!ready) {
                Thread.yield();
            }
            System.out.println(number);
        }
    }
    public static void main(String[] args) {
        new ReaderThread().start();
        number = 42;
        ready = true;
    }
}
```

We expect the `ReaderThread` prints 42. But it turns out the thread could loop forever or event print 0.

It could loop forever because in absence of `synchronized` change made in the main thread could be invisible to `ReaderThread`, causing inifinite loop. To address this problem, always put lock to the places write and read the shared variable.

It could print 0, because without `synchronized`, order of `number = 42` and `ready=true` could be reversed. In java, there's no guarantee that operations in one thread will be executed in order if the order change is invisible to the current thread, even this reordering might effect other thread. So if some thread depends on the order of operations in another thread, always put those operations in `synchronized` block.

### volatile variable
If a variable is declared as `volatile`, the compiler and runtime will ensure the operations on this variable will not be reordered with other memory operations, and the variable should not be cached in register where they will not be visible from other processors.

Volatile also has another meaning: when thread A writes a volatile variable, all the variables that are visible to A before the writes will also be visible to thread B after it reads the same variable. eg.
```java
public class MyClass {
    private int years;
    private int months
    private volatile int days;

    public int totalDays() {
        int total = this.days;
        total += months * 30;
        total += years * 365;
        return total;
    }

    public void update(int years, int months, int days){
        this.years  = years;
        this.months = months;
        this.days   = days;
    }
}
```
In this program, even `years` and `months` are not volatile, since their writes happen before writes of `days`, and their reads happen after reads of `days`, they're guranteed to be visible with their most recent value.

Volatile ensures ordering of operations with following rules:
- read/write of other variables can't be rearraged after writes of volatile variable if they were originally before the writes of that variable. Note if the read/write of other variables were originally after the writes of volatile variable, they're still able to be rearranged before writes of volatile variable.
- read/write of other variable can't be rearranged before reads of volatile variable if they were originally after the reads. Similarly, if they were before reads of volatile variable, they are still free to go after the reads.

Of course, volatile does not gurantee the atomicity, it also ensures variable is visible to other threads. To ensure both visiblity and atomicity, lock is still required.

## Publication and escape
Publishing an object means making it avaialble to code outside the current scope.
For example, returning a reference of internal state from a non-private method will expose that state to outside world.

### Several cases of publication:
#### public static field
publish static field is always visible to anywhere
#### returning an instance of inner class
instance of inner class holds this from outter class. By returnning the instance, it effectively published the containner classs.
#### Publishing an object also publishes its non private reference to other objects

### Do not let this reference escape during construction
If `this` escape from constructor, it is considered an incomplete object, even the escape happens in the last statement of constructor.

## Thread confinement
By restricting the data inside thread, there would be no share of data between threads, so that thread safey can be assured.
### Stack confinment
If thread only reflies on data of local variables, as the data only exists on thread stack, it is not shared, so it's thread safe.
### ThreadLocal
`ThreadLocal` is a data structure, which holds a separate copy of data for each thread.

## Immutability
Immutable objects are always thread-safe.

A object is immutable if:
- its state cannot be changed after construction
- All its fields are final; and
- it its properly constructed(`this` reference is not escaped during construction)

Note making all the fields final won't make an object immutable, as the field may reference to mutable object.

## Safe publication
If we do need to share data among threads, we need to be sure the data is published safely. Simply returns a reference to an object is NOT safe. Because the change of data might not visible to other thread.

### Sefe publication idiom
A properly constructed object can be safely published by:
- initializing an object reference from a static initializer
- storing a reference to it into a volatile field or AtomicReference.
- storing a reference to it into a final field of a properly constructed object
- storing a reference to it into a field that is guarded by lock

The requirement for safe publication depends on the mutability:

For immutable objects, they can be published by any means.

For Effectively immutable objects(objects which are not immutable, but logically will not be modified), objects must be safely published.

For Mutable objects, they must be safely published, and must be either thread-safe or guarded by a lock.