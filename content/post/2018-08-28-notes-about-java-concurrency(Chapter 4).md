---
title: "Notes about java concurrency"
categories: ["tech"]
date: 2018-08-28
tags: ["java"]
---

# Composing Objects
This is about how to compose a large thread-safe components from smaller thread-safe components.


## Designing a thread-safe class
To design a thread-safe class, one needs to first find out its invariant, and makes sure the invariant holds in all concurrent cases. For example, if the invariant is there's no duplicate in the list, then we need to make sure we don't end up inserting duplicate entry in concurrent envrionment.

Some operations of a class is state-dependent. For instance, we can't remove element from an empty list. In concurrent environment, even the list is empty, and we try to remove an element, we have the option to wait until there's an element in the list because other threads may insert something into the list. This is different with single threaded program.

Also there's ownership. A collecction owns its collection infrastructure, but not the objects stored in the collection. Even with thread safe collections, we still need to syncronization when using the object in the collection.

## Java monitor pattern
An object follows Java monitor pattern encapsulates all its mutable state and guard it with object's own instrinsic lock. A private lock can serve the same purpose.
This counter example is the simplist Java monitor pattern:
```java
public class Counter {
    int count = 0;
    public synchronized int inc() {
        count++;
        return count;
    }
}
```

## Delegating thread safety
If an object's composing objects are thread-safe, the object is not necessarily a thread-safe component.

If the object contains only one thread-safe variable, or several independent thread-safe variable, the object can delegate the thread-safety to these variables. But if the variables interact with eachother, the resulting object is usually not thread-safe.

## Adding functionality to existing thread-safe classes
### extends the class
To add new functionality to an existing thread-safe class, we can extend the original class, and add new syncronized function. But this is not good, because we're assuming the syncronization policy of the parent class. If parent class changes its locking policy, like changing from syncronization to private lock, the extended class will break.

Example of adding a `putIfAbsent` method to Vector:
```java
public class BetterVector<E> extends Vector<E> {
    public synchronized boolean putIfAbsent(E x) {
        boolean absent = !contains(x);
        if (absent) {
            this.put(x);
        }
    }
}
```
We're assuming the Vector is using intrinsic lock, but this is not a guarantee. This will make out code fragile to change.

### Client side locking
Client side locking wrap the thread-safe object in another class, and implement the method in wrapper class, and do lock there:
```java
public class ListHelper<E> {
    public List<E> list = Collections.synchronizedList(new ArrayList<E>());
    public boolean putIfAbsent(E x) {
        synchronized (list) {
            boolean absent = !list.contains(x); if (absent)
        }
    }
}
```
Note we can't write the method like: `public synchronzied boolean putIfAbsent`, becuase this will lock on the wrapper class, making this method guarded by different lock as the other methods of wrapped class. Leaving this method not thread-safe with other methods.

### Composition
A better way is to make a class implementing the interface of another class, and delegate all the methods to another class. And use our own lock to guard every method. This way, we don't assume any information about the lock mechanism of another class.

```java
public class ImprovedList<T> implements List<T> {
    private final List<T> list;
    public ImprovedList(List<T> list) { this.list = list; }
    public synchronized boolean putIfAbsent(T x) {
        boolean contains = list.contains(x);
        if (contains)
            list.add(x);
        return !contains;
    }
    public synchronized void clear() { list.clear(); }
    // ... similarly delegate other List methods
}
```
