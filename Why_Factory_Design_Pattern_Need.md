In Java, factory design patterns are used to create objects without exposing the creation logic to the client code. This promotes encapsulation, loose coupling, and code reusability. There are several types of factory design patterns, but the most commonly used are:

1. **Factory Method Pattern**:
The Factory Method Pattern defines an interface for creating an object, but lets subclasses decide which class to instantiate. This pattern is useful when a class cannot anticipate the class of objects it must create.

Example:
Consider a scenario where we have a `LoggerFactory` interface and two implementations: `FileLogger` and `DatabaseLogger`. Depending on the configuration, we want to create either a `FileLogger` or a `DatabaseLogger` instance.

```java
// Logger interface
interface Logger {
    void log(String message);
}

// Concrete Logger implementations
class FileLogger implements Logger {
    public void log(String message) {
        System.out.println("Writing to file: " + message);
    }
}

class DatabaseLogger implements Logger {
    public void log(String message) {
        System.out.println("Writing to database: " + message);
    }
}

// LoggerFactory interface
interface LoggerFactory {
    Logger createLogger();
}

// Concrete LoggerFactory implementations
class FileLoggerFactory implements LoggerFactory {
    public Logger createLogger() {
        return new FileLogger();
    }
}

class DatabaseLoggerFactory implements LoggerFactory {
    public Logger createLogger() {
        return new DatabaseLogger();
    }
}

// Client code
public class Main {
    public static void main(String[] args) {
        LoggerFactory loggerFactory;
        String loggerType = "file"; // or "database"

        if (loggerType.equals("file")) {
            loggerFactory = new FileLoggerFactory();
        } else {
            loggerFactory = new DatabaseLoggerFactory();
        }

        Logger logger = loggerFactory.createLogger();
        logger.log("Hello, World!");
    }
}
```

2. **Abstract Factory Pattern**:
The Abstract Factory Pattern provides an interface for creating families of related or dependent objects without specifying their concrete classes.

Example:
Consider a scenario where we have a furniture store that sells chairs and tables of different types (e.g., modern, vintage).

```java
// AbstractFactory interface
interface FurnitureFactory {
    Chair createChair();
    Table createTable();
}

// Concrete factories
class ModernFurnitureFactory implements FurnitureFactory {
    public Chair createChair() {
        return new ModernChair();
    }

    public Table createTable() {
        return new ModernTable();
    }
}

class VintageFurnitureFactory implements FurnitureFactory {
    public Chair createChair() {
        return new VintageChair();
    }

    public Table createTable() {
        return new VintageTable();
    }
}

// Abstract products
interface Chair {
    void sitOn();
}

interface Table {
    void putOn();
}

// Concrete products
class ModernChair implements Chair {
    public void sitOn() {
        System.out.println("Sitting on a modern chair");
    }
}

class VintageChair implements Chair {
    public void sitOn() {
        System.out.println("Sitting on a vintage chair");
    }
}

class ModernTable implements Table {
    public void putOn() {
        System.out.println("Putting on a modern table");
    }
}

class VintageTable implements Table {
    public void putOn() {
        System.out.println("Putting on a vintage table");
    }
}

// Client code
public class Main {
    public static void main(String[] args) {
        FurnitureFactory furnitureFactory = new ModernFurnitureFactory();
        Chair chair = furnitureFactory.createChair();
        Table table = furnitureFactory.createTable();

        chair.sitOn();
        table.putOn();
    }
}
```

The factory design patterns promote loose coupling between the client code and the concrete object implementations. This makes the code more maintainable, extensible, and testable. The client code only needs to know about the abstract interfaces or base classes, not the specific implementations.