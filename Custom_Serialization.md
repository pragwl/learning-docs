Here's the detailed code documentation for the `CustomSerialization` example provided, formatted for study material. This documentation explains the purpose, functionality, and details of each section of the code to help learners understand how to perform custom serialization and deserialization in Java.

---

# Custom Serialization in Java

This study material covers custom serialization in Java, demonstrating how to selectively serialize and deserialize object fields using `writeObject` and `readObject` methods.

## Overview

In Java, the default serialization mechanism serializes all non-transient fields of an object. However, sometimes we need more control over the serialization process, such as when we want to serialize only specific fields or handle complex object structures. This can be achieved by implementing custom serialization methods.

## Example: Custom Serialization of `Employee` Class

In this example, we have a class `Employee` with three fields: `id`, `name`, and `salary`. We will serialize only the `id` and `name` fields, excluding `salary`.

### Code Structure

- **`CustomSerialization` Class**: Contains the `main` method, which demonstrates the serialization and deserialization process.
- **`Employee` Class**: A serializable class with custom `writeObject` and `readObject` methods to control serialization and deserialization.

### `CustomSerialization` Class

```java
import java.io.*;

// Main class to demonstrate custom serialization
public class CustomSerialization {
    public static void main(String[] args) {
        Employee employee = new Employee(1, "Priyank", 12500.0);
        System.out.println("Original Employee:\n" + employee);

        // Serialize the object
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("object.ser"))) {
            oos.writeObject(employee);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }

        Employee readFromStream;
        // Deserialize the object
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("object.ser"))) {
            readFromStream = (Employee) ois.readObject();
        } catch (IOException | ClassNotFoundException e) {
            throw new RuntimeException(e);
        }
        System.out.println("After read from stream : \n" + readFromStream);
    }
}
```

#### Explanation

1. **Creating an Employee Object**:
   - An `Employee` object is instantiated with `id`, `name`, and `salary`.

2. **Serialization**:
   - The `Employee` object is serialized to a file named `object.ser` using `ObjectOutputStream`.
   - Only the `id` and `name` fields are serialized, excluding the `salary` field.

3. **Deserialization**:
   - The `Employee` object is deserialized from the file `object.ser` using `ObjectInputStream`.
   - The deserialized object contains the `id` and `name`, but the `salary` is not restored as it was not serialized.

### `Employee` Class

```java
import java.io.*;

// Serializable class with custom serialization logic
class Employee implements Serializable {
    private Integer id;
    private String name;
    private transient Double salary; // Marked transient, won't be serialized by default

    // Constructor
    public Employee(Integer id, String name, Double salary) {
        this.id = id;
        this.name = name;
        this.salary = salary;
    }

    // Getters for the fields
    public Integer getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public Double getSalary() {
        return salary;
    }

    // Custom serialization method
    private void writeObject(ObjectOutputStream oos) throws IOException {
        // Serialize specific fields
        oos.writeInt(getId());
        oos.writeObject(getName());
        // Other field (salary) is not serialized
    }

    // Custom deserialization method
    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        // Read object fields from the stream
        this.id = in.readInt();
        this.name = (String) in.readObject();
        // The salary field is not deserialized and will be null or default value
    }

    // toString method for displaying the object's state
    @Override
    public String toString() {
        return "Employee{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", salary=" + salary +
                '}';
    }
}
```

#### Explanation

1. **Fields**:
   - `id`: An integer representing the employee's ID.
   - `name`: A string representing the employee's name.
   - `salary`: A double representing the employee's salary, marked as `transient` to indicate it should not be serialized by default.

2. **Custom Serialization (`writeObject`)**:
   - This method specifies which fields to serialize. In this case, only `id` and `name` are serialized using `writeInt` and `writeObject`.
   - The `salary` field is intentionally excluded from serialization.

3. **Custom Deserialization (`readObject`)**:
   - This method specifies how to deserialize the object. It reads the `id` and `name` from the input stream using `readInt` and `readObject`.
   - The `salary` field remains uninitialized or null since it is not part of the serialized data.

4. **toString Method**:
   - Provides a string representation of the `Employee` object, including all fields for display purposes.

## Summary

This example demonstrates how to perform custom serialization in Java by using the `writeObject` and `readObject` methods. By implementing these methods, you can:

- Control which fields are serialized and deserialized.
- Customize the serialization format.
- Ensure backward compatibility and handle complex serialization requirements.

Custom serialization is useful in scenarios where you need to exclude sensitive information, manage large data efficiently, or comply with specific data formats for interoperability.

---

This documentation provides a comprehensive guide on how custom serialization works in Java, including a detailed code example and explanation of each part. It serves as a useful reference for understanding and implementing custom serialization in Java applications.