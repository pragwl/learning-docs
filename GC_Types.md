Certainly. In Java, garbage collection (GC) is the process of automatically freeing memory by removing objects that are no longer in use. The three types of garbage collection you've asked about - minor, major, and full - differ in scope and impact on application performance. Let's explore each type:

1. Minor Garbage Collection:

Minor GC, also known as Young Generation GC, deals with the Young Generation of the heap.

Characteristics:
- Occurs frequently
- Usually very fast
- Causes short pauses in application execution

Example scenario:
```java
for (int i = 0; i < 1000000; i++) {
    new TemporaryObject();
}
```
In this case, many short-lived TemporaryObject instances are created. They'll likely be collected by minor GC.

Purpose: To quickly remove short-lived objects, keeping the Young Generation space available for new object allocation.

2. Major Garbage Collection:

Major GC, also known as Old Generation GC, deals with the Old Generation of the heap.

Characteristics:
- Less frequent than Minor GC
- Takes more time than Minor GC
- Causes longer pauses than Minor GC

Example scenario:
```java
List<LongLivedObject> list = new ArrayList<>();
for (int i = 0; i < 100000; i++) {
    list.add(new LongLivedObject());
}
```
Here, LongLivedObject instances survive multiple minor GCs and are promoted to the Old Generation. Eventually, a Major GC will be triggered to clean up these objects when they're no longer needed.

Purpose: To remove objects that have lived long enough to be promoted to the Old Generation but are now no longer in use.

3. Full Garbage Collection:

Full GC involves cleaning both the Young and Old Generations, and often includes cleaning the Metaspace (or PermGen in older Java versions).

Characteristics:
- Least frequent
- Takes the most time
- Causes the longest application pauses

Example scenario:
```java
// Application running for a long time
// Continuously allocating both short-lived and long-lived objects
// Eventually, both Young and Old generations fill up
System.gc(); // This can trigger a Full GC, though it's generally not recommended to call explicitly
```

Purpose: To clean up the entire heap when both Young and Old Generations are significantly full.

Here's a more concrete example that might lead to different types of GC:

```java
import java.util.ArrayList;
import java.util.List;

public class GCExample {
    public static void main(String[] args) {
        List<byte[]> list = new ArrayList<>();
        
        while (true) {
            // Allocate 1MB
            byte[] b = new byte[1024 * 1024];
            list.add(b);
            
            // Remove some objects to simulate turnover
            if (list.size() > 100) {
                for (int i = 0; i < 10; i++) {
                    list.remove(0);
                }
            }
            
            // This sleep is to prevent the program from consuming all CPU
            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

In this example:
1. Minor GC will occur frequently to clean up short-lived byte arrays that are removed from the list.
2. Major GC will occur less frequently to clean up longer-lived byte arrays that have been promoted to the Old Generation.
3. Full GC might occur if both Young and Old Generations fill up, or if the system is under memory pressure.

To observe these GCs, you can run this program with GC logging enabled:

```
java -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:gc.log GCExample
```

This will produce a gc.log file showing the different types of GC occurring over time.

Remember, the exact behavior depends on factors like the JVM version, GC algorithm used, heap size, and application characteristics. Modern JVMs have sophisticated GC algorithms that aim to minimize the impact of garbage collection on application performance.

Would you like me to elaborate on any specific aspect of garbage collection?