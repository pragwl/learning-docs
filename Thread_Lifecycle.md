Certainly. In Java, threads can exist in several states throughout their lifecycle. Let's go through each state and provide examples of when a thread might enter that state and for what purpose.

1. New
2. Runnable
3. Blocked
4. Waiting
5. Timed Waiting
6. Terminated

Let me explain each state with examples:

1. New:
   When a thread is created but not yet started.

   Example:
   ```java
   Thread t = new Thread(() -> System.out.println("Hello"));
   // The thread t is in NEW state here
   ```
   Purpose: To create a thread object before it's ready to run.

2. Runnable:
   When a thread is ready to run and waiting for CPU time.

   Example:
   ```java
   Thread t = new Thread(() -> System.out.println("Hello"));
   t.start();
   // The thread t is now in RUNNABLE state
   ```
   Purpose: The thread is either running or ready to run when it gets CPU time.

3. Blocked:
   When a thread is waiting to acquire a lock on an object.

   Example:
   ```java
   synchronized(sharedObject) {
       // Some code that takes time
   }
   // Another thread trying to enter this synchronized block will be BLOCKED
   ```
   Purpose: To ensure thread-safe access to shared resources.

4. Waiting:
   When a thread is waiting indefinitely for another thread to perform a particular action.

   Example:
   ```java
   synchronized(sharedObject) {
       sharedObject.wait();
   }
   // The thread is now in WAITING state
   ```
   Purpose: To coordinate actions between threads.

5. Timed Waiting:
   Similar to waiting, but with a specified maximum time to wait.

   Example:
   ```java
   Thread.sleep(1000);
   // The thread is now in TIMED_WAITING state for 1 second
   ```
   Purpose: To pause thread execution for a specific duration or wait for an event with a timeout.

6. Terminated:
   When a thread has completed its execution or was stopped.

   Example:
   ```java
   public void run() {
       System.out.println("Thread task completed");
   }
   // After this method completes, the thread is in TERMINATED state
   ```
   Purpose: To indicate that the thread has finished its work and its resources can be reclaimed.

Here's a more comprehensive example demonstrating multiple thread states:

```java
public class ThreadStatesDemo {
    public static void main(String[] args) throws InterruptedException {
        Object lock = new Object();

        Thread t = new Thread(() -> {
            try {
                synchronized(lock) {
                    System.out.println("Thread acquired lock");
                    lock.wait(2000);  // TIMED_WAITING
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        System.out.println("Thread state after creation: " + t.getState());  // NEW

        t.start();
        System.out.println("Thread state after starting: " + t.getState());  // RUNNABLE

        Thread.sleep(100);  // Give some time for the thread to acquire the lock
        System.out.println("Thread state while waiting: " + t.getState());  // TIMED_WAITING

        synchronized(lock) {
            lock.notify();
            System.out.println("Thread state after notification: " + t.getState());  // BLOCKED (waiting to reacquire lock)
        }

        t.join();
        System.out.println("Thread state after completion: " + t.getState());  // TERMINATED
    }
}
```

This example demonstrates how a single thread moves through different states during its lifecycle.

Would you like me to explain or break down any part of this code or concept further?