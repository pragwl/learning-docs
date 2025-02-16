Here's a comprehensive guide to the different scenarios of method inheritance in Java, focusing on both default and non-default methods. This should serve as a quick reference for interview preparation.

### **1. Single Interface with Default Method**

**Scenario:**  
A class implements a single interface with a default method.

```java
interface Interface1 {
    default void method() {
        System.out.println("Interface1 method");
    }
}

class Class1 implements Interface1 {
    // No override
}
```

**Explanation:**  
- `Class1` inherits the `method()` from `Interface1`.
- **Output when `method()` is called:**  
  ```
  Interface1 method
  ```

### **2. Multiple Interfaces with Different Default Methods**

**Scenario:**  
A class implements multiple interfaces, each with a different default method.

```java
interface Interface1 {
    default void method1() {
        System.out.println("Interface1 method1");
    }
}

interface Interface2 {
    default void method2() {
        System.out.println("Interface2 method2");
    }
}

class Class1 implements Interface1, Interface2 {
    // No override
}
```

**Explanation:**  
- `Class1` inherits both `method1()` and `method2()` from `Interface1` and `Interface2`.
- **Output when `method1()` or `method2()` is called:**
  ```
  Interface1 method1
  Interface2 method2
  ```

### **3. Multiple Interfaces with the Same Default Method**

**Scenario:**  
A class implements multiple interfaces that have the same default method.

```java
interface Interface1 {
    default void method() {
        System.out.println("Interface1 method");
    }
}

interface Interface2 {
    default void method() {
        System.out.println("Interface2 method");
    }
}

class Class1 implements Interface1, Interface2 {
    @Override
    public void method() {
        Interface1.super.method(); // or Interface2.super.method();
    }
}
```

**Explanation:**  
- `Class1` must override `method()` to resolve the conflict.
- You can explicitly call either `Interface1.super.method()` or `Interface2.super.method()` to choose which implementation to use.
- **Output depends on the choice:**  
  ```
  Interface1 method
  ```
  or
  ```
  Interface2 method
  ```

### **4. Interface Inherits Default Method from Another Interface**

**Scenario:**  
An interface inherits a default method from another interface.

```java
interface Interface1 {
    default void method() {
        System.out.println("Interface1 method");
    }
}

interface Interface2 extends Interface1 {
    // No override
}

class Class1 implements Interface2 {
    // No override
}
```

**Explanation:**  
- `Interface2` inherits `method()` from `Interface1`.
- `Class1` inherits `method()` from `Interface2` (which got it from `Interface1`).
- **Output when `method()` is called:**  
  ```
  Interface1 method
  ```

### **5. Interface Inherits and Overrides Default Method**

**Scenario:**  
An interface inherits a default method from another interface and overrides it.

```java
interface Interface1 {
    default void method() {
        System.out.println("Interface1 method");
    }
}

interface Interface2 extends Interface1 {
    @Override
    default void method() {
        System.out.println("Interface2 method");
    }
}

class Class1 implements Interface2 {
    // No override
}
```

**Explanation:**  
- `Interface2` overrides the default method from `Interface1`.
- `Class1` inherits the overridden method from `Interface2`.
- **Output when `method()` is called:**  
  ```
  Interface2 method
  ```

### **6. Non-Default Method in Interface and Default Method in Another Interface**

**Scenario:**  
A class implements one interface with a non-default method and another with a default method of the same name.

```java
interface Interface1 {
    void method();
}

interface Interface2 {
    default void method() {
        System.out.println("Interface2 method");
    }
}

class Class1 implements Interface1, Interface2 {
    @Override
    public void method() {
        System.out.println("Class1 method");
    }
}
```

**Explanation:**  
- `Class1` must provide an implementation for `method()` because `Interface1` has a non-default method (abstract method).
- The default method from `Interface2` is not used.
- **Output when `method()` is called:**  
  ```
  Class1 method
  ```

### **7. Inheriting and Overriding a Non-Default Method**

**Scenario:**  
A class inherits a non-default method from a class and overrides it.

```java
class Parent {
    void method() {
        System.out.println("Parent method");
    }
}

class Child extends Parent {
    @Override
    void method() {
        System.out.println("Child method");
    }
}
```

**Explanation:**  
- `Child` overrides the `method()` from `Parent`.
- **Output when `method()` is called:**  
  ```
  Child method
  ```

### **8. Class Implements Interface and Overrides Default Method**

**Scenario:**  
A class implements an interface with a default method but overrides it.

```java
interface Interface1 {
    default void method() {
        System.out.println("Interface1 method");
    }
}

class Class1 implements Interface1 {
    @Override
    public void method() {
        System.out.println("Class1 method");
    }
}
```

**Explanation:**  
- `Class1` overrides the default `method()` provided by `Interface1`.
- **Output when `method()` is called:**  
  ```
  Class1 method
  ```

### **9. Abstract Class with Abstract Method and Class Implements Interface**

**Scenario:**  
A class implements an interface with a default method and extends an abstract class with an abstract method of the same name.

```java
abstract class AbstractClass {
    abstract void method();
}

interface Interface1 {
    default void method() {
        System.out.println("Interface1 method");
    }
}

class Class1 extends AbstractClass implements Interface1 {
    @Override
    public void method() {
        System.out.println("Class1 method");
    }
}
```

**Explanation:**
- `Class1` must provide an implementation for the abstract method in `AbstractClass`, even though `Interface1` provides a default method.
- The abstract method takes precedence, so `Class1` must override it.
- **Output when `method()` is called:**  
  ```
  Class1 method
  ```

### **10. Concrete Class and Interface with Same Method Name**

**Scenario:**  
A class extends a concrete class and implements an interface that both have the same method name.

```java
class Parent {
    void method() {
        System.out.println("Parent method");
    }
}

interface Interface1 {
    default void method() {
        System.out.println("Interface1 method");
    }
}

class Child extends Parent implements Interface1 {
    // No override
}
```

**Explanation:**
- The method in the concrete `Parent` class takes precedence over the interface's default method.
- **Output when `method()` is called:**  
  ```
  Parent method
  ```

### **Summary of Key Points:**
- **Default Methods in Interfaces:** Allow for multiple inheritance-like behavior, but ambiguity must be resolved by the implementing class.
- **No Multiple Inheritance for Classes:** Java avoids the complexity and potential errors of multiple inheritance by only allowing single inheritance for classes.
- **Method Resolution Order:** For classes, concrete class methods take precedence over interface default methods. For interfaces, the most specific implementation (closest in the inheritance hierarchy) is chosen.

This guide should cover the major scenarios you'll encounter and help you prepare for interview questions related to method inheritance in Java.