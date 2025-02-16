Threads in Java can communicate with each other using various mechanisms provided by the Java concurrency utilities. Here are the main ways threads can communicate, along with examples:

1. **Shared Memory (Variables/Objects):**
Threads can communicate by sharing variables or objects and modifying their values or state. However, this approach requires proper synchronization to avoid race conditions and thread safety issues.

Example:
```java
public class SharedMemoryExample {
    private static int sharedVariable = 0;

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                sharedVariable++;
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                sharedVariable++;
            }
        });

        t1.start();
        t2.start();

        try {
            t1.join();
            t2.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("Final value of sharedVariable: " + sharedVariable);
    }
}
```

2. **Wait and Notify:**
The `wait()`, `notify()`, and `notifyAll()` methods are used for inter-thread communication when one thread needs to wait for a condition to be met by another thread. These methods are used in conjunction with synchronized blocks or methods.

Example:
```java
public class WaitNotifyExample {
    private static boolean messageAvailable = false;
    private static String message = null;

    public static void main(String[] args) {
        Object lock = new Object();

        Thread sender = new Thread(() -> {
            synchronized (lock) {
                message = "Hello, World!";
                messageAvailable = true;
                lock.notifyAll();
            }
        });

        Thread receiver = new Thread(() -> {
            synchronized (lock) {
                while (!messageAvailable) {
                    try {
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.println("Received message: " + message);
            }
        });

        sender.start();
        receiver.start();
    }
}
```

3. **Semaphores:**
Semaphores are used to control access to shared resources by multiple threads. They allow a limited number of threads to access a resource at a time.

Example:
```java
import java.util.concurrent.Semaphore;

public class SemaphoreExample {
    private static Semaphore semaphore = new Semaphore(3); // Maximum 3 threads can access at a time

    public static void main(String[] args) {
        for (int i = 1; i <= 10; i++) {
            Thread thread = new Thread(() -> {
                try {
                    semaphore.acquire();
                    System.out.println(Thread.currentThread().getName() + " is accessing the resource");
                    Thread.sleep(2000); // Simulate some work
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    semaphore.release();
                }
            });
            thread.start();
        }
    }
}
```

4. **CountDownLatch:**
A `CountDownLatch` is used to make one or more threads wait until a set of operations being performed in other threads completes.

Example:
```java
import java.util.concurrent.CountDownLatch;

public class CountDownLatchExample {
    public static void main(String[] args) {
        CountDownLatch latch = new CountDownLatch(3);

        Thread t1 = new Thread(() -> {
            System.out.println("Thread 1 is running");
            latch.countDown();
        });

        Thread t2 = new Thread(() -> {
            System.out.println("Thread 2 is running");
            latch.countDown();
        });

        Thread t3 = new Thread(() -> {
            System.out.println("Thread 3 is running");
            latch.countDown();
        });

        Thread mainThread = new Thread(() -> {
            try {
                latch.await(); // Main thread waits for other threads to complete
                System.out.println("All threads have completed");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        t1.start();
        t2.start();
        t3.start();
        mainThread.start();
    }
}
```

5. **CyclicBarrier:**
A `CyclicBarrier` is used to make a set of threads wait for each other to reach a common barrier point before proceeding with their execution.

Example:
```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierExample {
    public static void main(String[] args) {
        CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("All threads have completed their tasks"));

        Thread t1 = new Thread(() -> {
            System.out.println("Thread 1 is running");
            try {
                barrier.await();
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        });

        Thread t2 = new Thread(() -> {
            System.out.println("Thread 2 is running");
            try {
                barrier.await();
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        });

        Thread t3 = new Thread(() -> {
            System.out.println("Thread 3 is running");
            try {
                barrier.await();
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        });

        t1.start();
        t2.start();
        t3.start();
    }
}
```

These are the main mechanisms for inter-thread communication in Java. The choice of which mechanism to use depends on the specific requirements of your application, such as whether you need to control access to shared resources, wait for a set of operations to complete, or synchronize threads at specific points in their execution.