A memory leak in Java occurs when objects are no longer used by the application but are still held in memory, preventing them from being garbage collected. This leads to a gradual loss of available memory over time, potentially causing performance issues or even OutOfMemoryError if left unchecked.

Memory leaks affect garbage collection in the following ways:

1. Increased frequency of GC: As leaked objects accumulate, GC runs more often to try to free up space.
2. Longer GC pauses: With more objects to scan, each GC cycle takes longer.
3. Reduced effectiveness: Despite more frequent and longer GCs, memory usage continues to grow.
4. Eventually, OutOfMemoryError: If the leak continues, the application may run out of memory entirely.

Let's look at some examples of memory leaks and how they impact garbage collection:

1. Static Collections

```java
public class StaticCollectionLeak {
    private static final List<byte[]> leak = new ArrayList<>();
    
    public void addToLeak() {
        leak.add(new byte[1024 * 1024]); // Add 1MB
    }
    
    public static void main(String[] args) {
        StaticCollectionLeak instance = new StaticCollectionLeak();
        while (true) {
            instance.addToLeak();
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

In this example, objects are continuously added to a static list. Since the list is static, these objects are never eligible for garbage collection, leading to a memory leak.

Impact on GC: 
- Minor GCs will occur frequently but won't be able to reclaim the leaked memory.
- Major GCs will also be ineffective in reclaiming this memory.
- Eventually, the Old Generation will fill up, leading to longer and more frequent Full GCs.

2. Unclosed Resources

```java
import java.io.*;

public class ResourceLeak {
    public static void main(String[] args) {
        while (true) {
            try {
                BufferedReader reader = new BufferedReader(new FileReader("large_file.txt"));
                String line;
                while ((line = reader.readLine()) != null) {
                    // Process the line
                }
                // Reader is never closed
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

In this example, BufferedReader instances are created but never closed. Each instance holds onto system resources and memory.

Impact on GC:
- These objects might be small, but they prevent larger underlying buffers from being collected.
- Over time, this leads to increased memory usage and more frequent GC cycles.

3. Caching without Size Limits

```java
import java.util.HashMap;
import java.util.Map;

public class UnboundedCacheLeak {
    private static final Map<String, byte[]> cache = new HashMap<>();
    
    public static void addToCache(String key) {
        cache.put(key, new byte[1024 * 1024]); // 1MB per entry
    }
    
    public static void main(String[] args) {
        for (int i = 0; i < Integer.MAX_VALUE; i++) {
            addToCache("key" + i);
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

This example demonstrates an unbounded cache that keeps growing without any eviction policy.

Impact on GC:
- The cache will keep growing, filling up the Old Generation.
- GC cycles will become longer and more frequent as the cache grows.
- Eventually, this will lead to OutOfMemoryError.

To observe the impact of these memory leaks on garbage collection, you can run these examples with GC logging enabled:

```
java -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xmx512m -Xloggc:gc.log LeakingClass
```

As the program runs, you'll observe in gc.log:
1. Increasing frequency of GC events.
2. Longer duration for each GC event.
3. Growing heap usage after each GC.
4. Eventually, very long Full GC times or OutOfMemoryError.

To prevent memory leaks:
1. Always close resources like file handles and database connections.
2. Use weak references or bounded caches for caching mechanisms.
3. Be cautious with static collections and long-lived objects.
4. Use memory profiling tools to identify and fix leaks.
5. Consider using try-with-resources for AutoCloseable resources.

By preventing memory leaks, you ensure that garbage collection can effectively reclaim memory, maintaining application performance and stability.

Would you like me to elaborate on any specific aspect of memory leaks or their impact on garbage collection?