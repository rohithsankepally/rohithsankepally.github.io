---
layout: post
title: Inter Process Locking Framework
blog : true
published: true
date: May 25, 2021
comments: true
---

This blog post deals with inter process locking mechanism and its advantages. It also includes an implementation of a distributed locking mechanism architecture using [Apache Curator](https://curator.apache.org/).

## Why locking?

Locking is a mechanism used to restrict access to shared object in memory. This maintains controlled access of the shared object being used by multiple threads. 

Let’s say a process is running two parallel threads **A** and **B** using a shared object **O**. Let’s say thread **A** wants to modify object **O**. **A** will first lock the object **O** and therefore thread **B** cannot access the object **O** for any kind of operation. Once thread **A** has completed its operation, it will release the lock on object **O**, thereby allowing thread **B** to resume and perform its operations. 

Locking helps maintain consistency. For instance let us re-think the same situation without lock. Let’s say thread **A** first modified object **O** without locking. Let’s call this update **U1**. Since **O** is not locked, thread **B** has made another update **U2** (before **U1** was applied).  Ideally, a consistent state would be that in thread **B** update **U1** should be reflected before **U2** Is made. But since, these threads are running in parallel there are high chances that thread **B** reads object **O** before **U1** was applied, thereby leading to inconsistency. If we rethink the scenario by using locking object **O** before making any update we can maintain a consistency across threads. Locking is very useful in programs with high concurrency.

## How to use locking in Java?

There are different ways of locking supported by Java. You can find more about locking in java [here](http://tutorials.jenkov.com/java-concurrency/locks.html). 

##  Distributed Locking

Till now we discussed about locking and how it helps in dealing with concurrency issues among threads. But sometimes concurrency can also be an issue across processes. Let’s say there is a object (being fetched from database) used by multiple processes running on different machines. Assume, every process tries to access that object makes a series of operations and saves that object back to the database. Now suppose, we want the updates to this object to be consistent i.e at most one thread across machines should modify this object at any given point of time. If we analyse this carefully, this is exactly the same scenario that discussed previously, but the threads are now distributed across multiple machines. To achieve this we would need a  distributed locking mechanism which maintains consistency by allowing us to acquire/release locks from different machines. 

## Inter Process Locking Framework With Apache Curator

[Apache Curator](https://curator.apache.org/) is a high level API framework written on top of [Apache Zookeeper](https://zookeeper.apache.org/) to improve its usability.

[Apache Curator](https://curator.apache.org/) supports various [locking mechanisms](https://curator.apache.org/curator-recipes/index.html). In this blog we will be using [Inter Process Mutex](https://curator.apache.org/curator-recipes/shared-reentrant-lock.html) and see how to develop a generic and usable distributed locking framework.

There are two important components of this framework

### DistributedLock
* An object corresponding to the entity which we want to achieve lock upon.
* Supports **lock()** function which is used to acquire a lock on the entity. Throws exception is  a lock cannot be acquired.
* Supports an **unlock()** function which releases lock acquired on the entity.
* You can find its complete implementation [here](https://github.com/rohithsankepally/apache-inter-process-lock/blob/master/src/main/java/DistributedLockImpl.java)  
 

### DistributedLockFactory
* Factory which provides us **DistributedLock** object given a key (for example the ID of the entity).
* Creates **DistributedLock** object when asked for & caches the lock object for further use.
* You can find its complete implementation [here](https://github.com/rohithsankepally/apache-inter-process-lock/blob/master/src/main/java/DistributedLockFactoryImpl.java)

### Hands-On
Lets go through the below unit test to understand the working of this framework.

<script src="https://gist.github.com/rohithsankepally/e751f7378143a196f4e485f30730f0ef.js"></script>

* On line 2, we fetch **DistributedLock** object for a given key (say ID of the entity).

* On line 4, we acquire the lock corresponding to this key.

* In lines 5-13, we  define the implementation of another thread say **Thread 2**. This thread tries to acquire the lock using the same key that was locked previously (in the main thread).

* On line 15, we start **Thread 2**. So, when **Thread 2** runs it fails to acquire lock as lock corresponding to this key has been acquired already (in the main thread).
   
You can find the full implementation of the above discussed unit test [here](https://github.com/rohithsankepally/apache-inter-process-lock/blob/master/src/test/java/DistributedLockUnitTest.java).


## Conclusion
* Locks are used to maintain consistency across threads
* To maintain consistency  across threads running on different machines a distributed locking framework will be needed.
* [Apache Curator](https://curator.apache.org/) provides different kinds of locking mechanisms that can be used for distributed workloads.
* You can find the full implementation of the locking framework discussed in this post [here](https://github.com/rohithsankepally/apache-inter-process-lock).

