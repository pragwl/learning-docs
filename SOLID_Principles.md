The SOLID principles are a set of design principles in object-oriented programming that help make software more understandable, flexible, and maintainable. Here's a simplified explanation of each principle with examples that you can use in an interview:

### 1. **Single Responsibility Principle (SRP)**
**Definition:** A class should have only one reason to change, meaning it should have only one responsibility or job.

**Example:**
Imagine you have a class `Invoice` that handles the calculation of the invoice amount, printing the invoice, and saving it to a database. This violates SRP because the class has multiple responsibilities. 

**Refactored:**
```java
class Invoice {
    public double calculateTotal() {
        // Calculation logic
    }
}

class InvoicePrinter {
    public void print(Invoice invoice) {
        // Printing logic
    }
}

class InvoiceRepository {
    public void save(Invoice invoice) {
        // Database saving logic
    }
}
```
Now, each class has a single responsibility: `Invoice` for calculations, `InvoicePrinter` for printing, and `InvoiceRepository` for database operations.

### 2. **Open/Closed Principle (OCP)**
**Definition:** Software entities (classes, modules, functions, etc.) should be open for extension but closed for modification.

**Example:**
Suppose you have a `Shape` class that calculates the area for different shapes like circles and rectangles. If you want to add a new shape (e.g., a triangle), you might be tempted to modify the existing `Shape` class. This violates OCP.

**Refactored:**
```java
abstract class Shape {
    abstract double area();
}

class Circle extends Shape {
    private double radius;
    public Circle(double radius) {
        this.radius = radius;
    }

    @Override
    double area() {
        return Math.PI * radius * radius;
    }
}

class Rectangle extends Shape {
    private double width, height;
    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }

    @Override
    double area() {
        return width * height;
    }
}
```
Now, if you want to add a new shape, you simply create a new class that extends `Shape` without modifying the existing code.

### 3. **Liskov Substitution Principle (LSP)**
**Definition:** Subtypes must be substitutable for their base types without altering the correctness of the program.

**Example:**
Consider a `Bird` class with a method `fly()`. If you create a subclass `Penguin` that overrides `fly()` to throw an exception (since penguins can't fly), it violates LSP because a `Penguin` cannot fully substitute a `Bird`.

**Refactored:**
```java
abstract class Bird {
    abstract void move();
}

class Sparrow extends Bird {
    @Override
    void move() {
        fly();
    }

    void fly() {
        // Flying logic
    }
}

class Penguin extends Bird {
    @Override
    void move() {
        walk();
    }

    void walk() {
        // Walking logic
    }
}
```
Now, `Penguin` and `Sparrow` both correctly override the `move()` method according to their own abilities, adhering to LSP.

### 4. **Interface Segregation Principle (ISP)**
**Definition:** Clients should not be forced to depend on interfaces they do not use. Instead of having a single large interface, it’s better to have multiple smaller, specific interfaces.

**Example:**
If you have an interface `Worker` with methods `work()` and `eat()`, and you create a `Robot` class that implements `Worker`, the `Robot` class might not need the `eat()` method.

**Refactored:**
```java
interface Workable {
    void work();
}

interface Eatable {
    void eat();
}

class HumanWorker implements Workable, Eatable {
    @Override
    public void work() {
        // Work logic
    }

    @Override
    public void eat() {
        // Eat logic
    }
}

class RobotWorker implements Workable {
    @Override
    public void work() {
        // Work logic
    }
}
```
Now, `HumanWorker` implements both `Workable` and `Eatable`, while `RobotWorker` only implements `Workable`, adhering to ISP.

### 5. **Dependency Inversion Principle (DIP)**
**Definition:** High-level modules should not depend on low-level modules. Both should depend on abstractions. Also, abstractions should not depend on details; details should depend on abstractions.

**Example:**
If a `Project` class directly depends on a specific `BackendDeveloper` and `FrontendDeveloper` class, it violates DIP because the high-level `Project` module depends on low-level implementation classes.

**Refactored:**
```java
interface Developer {
    void develop();
}

class BackendDeveloper implements Developer {
    @Override
    public void develop() {
        writeJavaCode();
    }

    private void writeJavaCode() {
        // Java code
    }
}

class FrontendDeveloper implements Developer {
    @Override
    public void develop() {
        writeJavaScriptCode();
    }

    private void writeJavaScriptCode() {
        // JavaScript code
    }
}

class Project {
    private List<Developer> developers;

    public Project(List<Developer> developers) {
        this.developers = developers;
    }

    public void implement() {
        developers.forEach(Developer::develop);
    }
}
```
Now, `Project` depends on the `Developer` interface (an abstraction) rather than specific implementations, adhering to DIP.

These simplified examples should help you explain the SOLID principles clearly during an interview.