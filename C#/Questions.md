# 1. Difference Between Value Types and Reference Types in C#

In C#, types are divided into **value types** and **reference types** based on how they store and manage data in memory.

---

## 🧩 1. Definition

### ✅ Value Types
- Store the **actual data** directly.
- Each variable **has its own copy** of the data.
- Typically stored in the **stack** memory.
- Faster access but limited in flexibility.

### ✅ Reference Types
- Store a **reference (memory address)** to the actual data.
- Multiple variables can **point to the same object**.
- The actual data is stored in the **heap** memory.
- More flexible but slightly slower to access.

---

## 🧠 2. Examples

| Type Category | Examples |
|----------------|-----------|
| **Value Types** | `int`, `float`, `bool`, `char`, `struct`, `enum` |
| **Reference Types** | `string`, `class`, `interface`, `array`, `delegate`, `object` |

---

## ⚙️ 3. Code Example

### 🔹 Value Type Example

```csharp
int x = 10;
int y = x;
y = 20;

Console.WriteLine(x); // Output: 10
Console.WriteLine(y); // Output: 20
```
## ⚙️ 3. Code Example

### 🔹 reference Type Example
```csharp
class Person
{
    public string Name { get; set; }
}

Person p1 = new Person { Name = "Alice" };
Person p2 = p1;
p2.Name = "Bob";

Console.WriteLine(p1.Name); // Output: Bob
Console.WriteLine(p2.Name); // Output: Bob
```

| Feature               | Value Type                       | Reference Type             |
| --------------------- | -------------------------------- | -------------------------- |
| **Memory Location**   | Stack                            | Heap                       |
| **Stores**            | Actual data                      | Memory address (reference) |
| **Assignment**        | Creates a copy                   | Copies the reference       |
| **Null Value**        | Cannot be null (unless nullable) | Can be null                |
| **Garbage Collector** | Not managed by GC                | Managed by GC              |
| **Performance**       | Faster                           | Slightly slower            |
| **Examples**          | `int`, `bool`, `struct`          | `class`, `string`, `array` |


### Notes
* `string` is a reference type, but it behaves like a value type because it is immutable (cannot be modified once created).

* Structs are value types, even though they can contain fields of reference types.

* Boxing and unboxing occurs when converting between value types and reference types.


# 2. Difference Between Boxing and Unboxing in C#

In C#, **boxing** and **unboxing** are processes used to convert **value types** to **reference types** and vice versa.  
They are essential concepts for understanding how the **.NET runtime (CLR)** handles memory and type conversions.

---

## 🧩 1. Definition

### ✅ Boxing
- **Boxing** is the process of **converting a value type** (like `int`, `bool`, `struct`) **into an object (reference type)**.
- The value is **copied from the stack** to the **heap**.
- This allows value types to be treated as objects.

```csharp
int num = 10;
object obj = num; // Boxing
```

### ✅ Unboxing

- Unboxing is the process of extracting **the value type from the boxed object**.
- The value is **copied from the heap** back to the **stack**.
- Requires explicit casting.

```csharp
object obj = 10;
int num = (int)obj; // Unboxing
```
### ⚙️ 2. Visual Representation
| Memory Area | Operation     | Description                                                       |
| ----------- | ------------- | ----------------------------------------------------------------- |
| 🧠 Stack    | Before Boxing | Holds the actual value of the value type                          |
| 💾 Heap     | After Boxing  | Stores the boxed object (reference to value)                      |
| 🔁 Unboxing | Reverse       | Retrieves the value from the heap and copies it back to the stack |

### 📊 3. Key Differences Between Boxing and Unboxing

| Feature               | Boxing                               | Unboxing                             |
| --------------------- | ------------------------------------ | ------------------------------------ |
| **Definition**        | Converts value type → reference type | Converts reference type → value type |
| **Direction**         | Stack → Heap                         | Heap → Stack                         |
| **Casting**           | Implicit                             | Explicit                             |
| **Performance**       | Slower (allocates new memory)        | Slower (type checking & copying)     |
| **Example**           | `object obj = num;`                  | `int num = (int)obj;`                |
| **Memory Allocation** | Creates a new object on the heap     | Extracts value from the heap         |

### 🚀 4. Performance Impact
- Boxing and unboxing are computationally expensive operations.
- Avoid excessive boxing/unboxing in performance-critical code (e.g., loops, collections).
- Use generics (e.g., List<int>) instead of non-generic collections (ArrayList) to prevent implicit boxing.

# 📘 3. Different Class Types in C#

In C#, classes are the **blueprints** for creating objects. The language provides various **types of classes** to support different programming scenarios.

---

## 1. **Concrete Class**
A **concrete class** is the most common type of class that can be **instantiated** to create objects.

### ✅ Example:
```csharp
public class Car
{
    public void StartEngine()
    {
        Console.WriteLine("Engine started!");
    }
}

// Usage
Car car = new Car();
car.StartEngine();
```
### 💡 Key Points:
- Can be instantiated directly.
- Can contain fields, methods, and properties.
- Does not need to be inherited.

## 2. **Abstract Class**

An **abstract class** provides a base definition for other classes and cannot be instantiated directly.

### ✅ Example:
```csharp
public abstract class Shape
{
    public abstract double CalculateArea(); // Abstract method
}

public class Circle : Shape
{
    public double Radius { get; set; }
    public override double CalculateArea() => Math.PI * Radius * Radius;
}
```
### 💡Keypoints
- Use the abstract keyword.
- May contain abstract and non-abstract members.
- Must be inherited to be used.

## 3. Sealed Class
A **sealed class** cannot be inherited by any other class.

### ✅ Example:
```csharp
public sealed class Logger
{
    public void Log(string message)
    {
        Console.WriteLine($"Log: {message}");
    }
}
```
### 💡Keypoints
- Use the `sealed` keyword.
- Improves performance by preventing inheritance hierarchy.
- Often used for security or utility classes.

## 4. Static Class
A **static class** contains only **static members** and **cannot be instantiated or inherited**.

### ✅ Example:
```csharp
public static class MathHelper
{
    public static int Add(int a, int b) => a + b;
}

```
### 💡Keypoints
- Use the static keyword.
- Members are accessed using the class name.
- Commonly used for utility or helper methods.

## 5. Partial Class
A partial class allows its definition to be split across multiple files.

### ✅ Example:
#### file1.cs
```csharp
public partial class Employee
{
    public void Work() => Console.WriteLine("Working...");
}

```
#### file2.cs
```csharp
public partial class Employee
{
    public void TakeBreak() => Console.WriteLine("Taking a break...");
}


```
### 💡Keypoints
- Use the `partial` keyword.
- All parts must use the same `access modifier` and `namespace`.
- Useful for large projects or auto-generated code (e.g., in ASP.NET).

## 6. Nested Class
A nested class is declared inside another class and is used when the inner class is only relevant to the outer class.

### ✅ Example:
```csharp
public class Outer
{
    public class Inner
    {
        public void Display() => Console.WriteLine("I am a nested class!");
    }
}
```
### 💡Keypoints
- Can be private, protected, or public.
- Accessed using `Outer.Inner` syntax.
- Used to logically group related functionality.

## 🧠 Summary Table
| Class Type | Can Instantiate | Can Inherit | Contains Static Members | Use Case                 |
| ---------- | --------------- | ----------- | ----------------------- | ------------------------ |
| Concrete   | ✅ Yes           | ✅ Yes       | ✅                       | Regular objects          |
| Abstract   | ❌ No            | ✅ Yes       | ✅                       | Base for derived classes |
| Sealed     | ✅ Yes           | ❌ No        | ✅                       | Prevent inheritance      |
| Static     | ❌ No            | ❌ No        | ✅ Only static           | Utility functions        |
| Partial    | ✅ Yes           | ✅ Yes       | ✅                       | Split class definition   |
| Nested     | ✅ Yes           | ✅ Yes       | ✅                       | Group related logic      |

# 🧭 4. Namespace in C#

In C#, a **namespace** is used to **organize and group related classes, interfaces, structs, enums, and other types** together.  
It helps avoid **naming conflicts** and makes the code **more maintainable and readable**.

---

## 🧠 What is a Namespace?

A **namespace** acts as a **container** for types and defines a **scope** that helps organize large codebases logically.

### ✅ Syntax:
```csharp
namespace MyApplication.Models
{
    public class Employee
    {
        public string Name { get; set; }
    }
}
```

## 💡 Key Benefits:

1. **Avoids naming conflicts**
   - Two classes with the same name can exist in different namespaces.

2. **Improves code organization**

   - Groups related functionalities together.

3. **Enhances readability**

   - Makes it easier to locate and maintain code.

4. **Supports modular programming**

   - Encourages logical separation of components.

## 📦 Accessing Namespaces

You can access types from another namespace `using` the using directive.

```csharp
using MyApplication.Models;

namespace MyApplication.Services
{
    public class EmployeeService
    {
        public void PrintEmployee()
        {
            Employee emp = new Employee { Name = "John" };
            Console.WriteLine(emp.Name);
        }
    }
}
```
## 🌳 Nested Namespaces

Namespaces can be nested to represent hierarchical structures.

```csharp
namespace Company
{
    namespace HR
    {
        public class Employee { }
    }
}

//equivalent modern syntax
namespace Company.HR
{
    public class Employee { }
}


```

## 🧩 Aliases in Namespace

You can create an alias for a namespace to simplify long names or resolve naming conflicts.

```csharp
using ProjModels = ProjectA.Models;

ProjModels.Employee emp = new ProjModels.Employee();

```
## 🧠 Summary
| Concept        | Description                                    |
| -------------- | ---------------------------------------------- |
| **Definition** | Logical grouping of related types.             |
| **Keyword**    | `namespace`                                    |
| **Purpose**    | Avoid naming conflicts, improve organization.  |
| **Access**     | Via `using` directive or fully qualified name. |
| **Example**    | `System.Collections.Generic.List<T>`           |

# 🧩 5. Difference Between Abstract Class and Interface in C#

In C#, both **Abstract Classes** and **Interfaces** are used to achieve **abstraction** — hiding implementation details and exposing only the required functionality.  
However, they serve different design purposes and have key distinctions.

---

## 🧠 What is an Abstract Class?

An **abstract class** is a **base class** that cannot be instantiated.  
It may contain both **abstract members** (no implementation) and **concrete members** (with implementation).

### ✅ Example
```csharp
public abstract class Shape
{
    public abstract double CalculateArea(); // Abstract method

    public void Display()
    {
        Console.WriteLine("This is a shape.");
    }
}

public class Circle : Shape
{
    public double Radius { get; set; }
    public override double CalculateArea() => Math.PI * Radius * Radius;
}
```
## 💡 Key Points

- Declared using the abstract keyword.
- Can include fields, properties, methods, and constructors.
- Supports access modifiers (public, protected, etc.).
- Useful when classes share a common base or behavior.

## 🧩 What is an Interface?

An interface defines a contract that a class or struct must follow.
It contains only declarations of members — implementation is provided by the class.

```csharp
public interface IShape
{
    double CalculateArea();
}

public class Rectangle : IShape
{
    public double Width { get; set; }
    public double Height { get; set; }

    public double CalculateArea() => Width * Height;
}

```
## 💡 Key Points

- Declared using the interface keyword.
- Cannot have fields or constructors.
- All members are public by default.
- A class can implement multiple interfaces.
- From C# 8.0 onwards, interfaces can have default implementations.
## ⚖️ Key Differences

| Feature                  | Abstract Class                                | Interface                                        |
| ------------------------ | --------------------------------------------- | ------------------------------------------------ |
| **Keyword**              | `abstract`                                    | `interface`                                      |
| **Implementation**       | Can have abstract and non-abstract methods    | Only declarations (C# 8+ allows default methods) |
| **Fields**               | Allowed                                       | Not allowed                                      |
| **Constructors**         | Allowed                                       | Not allowed                                      |
| **Access Modifiers**     | Supports all                                  | Members are `public` by default                  |
| **Multiple Inheritance** | A class can inherit only one abstract class   | A class can implement multiple interfaces        |
| **Use Case**             | For related classes sharing behavior or state | For defining a contract or capability            |
| **Versioning**           | Harder to extend                              | Easier (with default implementations)            |

## 🧠 Summary Table
| Concept                    | Abstract Class                | Interface                                         |
| -------------------------- | ----------------------------- | ------------------------------------------------- |
| **Instantiation**          | ❌ Not possible                | ❌ Not possible                                    |
| **Implementation Sharing** | ✅ Yes                         | ❌ No (except default methods)                     |
| **Fields/State**           | ✅ Allowed                     | ❌ Not allowed                                     |
| **Inheritance**            | Single                        | Multiple                                          |
| **Use When**               | You need shared code or state | You need a common contract across unrelated types |

# ⚙️6.  Difference Between `const` and `readonly` in C#

In C#, both **`const`** and **`readonly`** are used to declare **immutable (unchangeable)** fields,  
but they differ in **when and how** their values are assigned and stored.

---

## 🧠 What is `const`?

A **`const` (constant)** field is a **compile-time constant**, meaning its value must be **known and assigned at compile time**.

### ✅ Example
```csharp
public class MathConstants
{
    public const double Pi = 3.14159;
}
```
## 💡 Key Points

- Must be initialized at the time of declaration.

- Value is fixed at `compile time` — cannot be changed later.

- Automatically treated as `static (shared across all instances)`.

- Can only be primitive types, enum, or string.

- Cannot be assigned dynamically (e.g., inside constructors).

## ⚠️ Limitation Example
```csharp
public class Example
{
    public const int Value = GetValue(); // ❌ Error: must be constant
    private static int GetValue() => 10;
}

```

## 🧩 What is readonly?

A readonly field’s value is assigned at `runtime`, either:

1. At declaration, or

2. Inside a constructor.

## ✅ Example
```csharp
public class Circle
{
    public readonly double Radius;

    public Circle(double radius)
    {
        Radius = radius; // Allowed - runtime assignment
    }
}

```

## 💡 Key Points

- Value can be set `once at runtime (in constructor)`.

- Can hold runtime-computed values.

- Not implicitly static — can be instance or static.

- Useful for values that vary per instance but should remain immutable after initialization.

## ⚖️ Key Differences Between const and readonly
| Feature                 | `const`                                           | `readonly`                                       |
| ----------------------- | ------------------------------------------------- | ------------------------------------------------ |
| **Initialization Time** | Compile time                                      | Runtime (in constructor or declaration)          |
| **Value Assignment**    | Must be assigned at declaration                   | Can be assigned at declaration or in constructor |
| **Mutability**          | Immutable (compile-time)                          | Immutable after runtime initialization           |
| **Implicit Modifier**   | Implicitly `static`                               | Can be instance-level or static                  |
| **Data Types Allowed**  | Only primitive types, enums, and strings          | Any data type (including reference types)        |
| **Memory Allocation**   | Stored in **metadata** (common for all instances) | Stored per **instance** (unless static)          |
| **Performance**         | Slightly faster (resolved at compile time)        | Slightly slower (resolved at runtime)            |
| **Example Use Case**    | Mathematical constants (`Pi`, `DaysInWeek`)       | Configuration values determined at runtime       |

## 🧩 Example Comparison
```csharp
public class Example
{
    public const int ConstValue = 100;           // Must be compile-time constant
    public readonly int ReadonlyValue;           // Set at runtime

    public Example(int val)
    {
        ReadonlyValue = val;                     // Allowed only here
    }
}

// Usage
var obj = new Example(500);
Console.WriteLine(Example.ConstValue);           // Output: 100
Console.WriteLine(obj.ReadonlyValue);            // Output: 500

```
## 🧠 Summary
| Concept                       | `const`         | `readonly`               |
| ----------------------------- | --------------- | ------------------------ |
| **When assigned?**            | Compile time    | Runtime                  |
| **Can be changed later?**     | ❌ No            | ❌ No (after constructor) |
| **Can use constructor?**      | ❌ No            | ✅ Yes                    |
| **Implicitly static?**        | ✅ Yes           | ❌ No                     |
| **Supports reference types?** | ❌ No            | ✅ Yes                    |
| **Best For**                  | Fixed constants | Immutable runtime values |

# 🔄 7. Difference Between `ref`, `out`, and `in` Parameters in C#

In C#, **`ref`**, **`out`**, and **`in`** are **parameter modifiers** that allow you to **pass arguments by reference** instead of by value.  
This means the called method can access and, in some cases, modify the original variable from the caller.

---

## 🧩 1. `ref` Keyword

The **`ref`** keyword is used to **pass an argument by reference**, allowing both the **caller and the callee** to modify the value.

### ✅ Example
```csharp
public void Increment(ref int number)
{
    number++;
}

int value = 5;
Increment(ref value);
Console.WriteLine(value); // Output: 6
```
## 💡 Key Points

- The variable must be initialized before being passed.
- Both input and output modifications are allowed.
- Any change made inside the method affects the original variable.

## 🧩 2. `out` Keyword

The `out` keyword is used to return multiple values from a method.

The called method must assign a value to an `out` parameter before returning.

### ✅ Example
```csharp
public void GetValues(out int x, out int y)
{
    x = 10;
    y = 20;
}

int a, b;
GetValues(out a, out b);
Console.WriteLine($"a = {a}, b = {b}"); // Output: a = 10, b = 20

```
## 💡 Key Points

- The variable does not need to be initialized before being passed.

- The method must assign a value before returning.

- Commonly used when a method needs to return multiple values.

## 🧩 3. `in` Keyword

The `in` keyword (introduced in C# 7.2) is used to pass arguments by reference,
but the method cannot modify the value — it’s `read-only`.

### ✅ Example
```csharp
public void Display(in int number)
{
    Console.WriteLine($"Value: {number}");
    // number++; ❌ Error: Cannot modify because it's passed as 'in'
}

int value = 50;
Display(in value);

```
### 💡 Key Points

- The variable must be initialized before being passed.

- The parameter is read-only inside the method.

- Helps avoid copying large structures while ensuring immutability.

- Improves performance for large value types (like struct).

## ⚖️ Comparison Table
| Feature                        | `ref`                   | `out`                  | `in`                               |
| ------------------------------ | ----------------------- | ---------------------- | ---------------------------------- |
| **Initialization Before Call** | ✅ Required              | ❌ Not required         | ✅ Required                         |
| **Must Assign in Method**      | ❌ No                    | ✅ Yes                  | ❌ No                               |
| **Can Modify Inside Method**   | ✅ Yes                   | ✅ Yes                  | ❌ No                               |
| **Pass by Reference**          | ✅ Yes                   | ✅ Yes                  | ✅ Yes                              |
| **Primary Use**                | Modify and return value | Return multiple values | Read-only performance optimization |
| **Introduced In**              | C# 1.0                  | C# 1.0                 | C# 7.2                             |

# ⚙️ 8. var vs dynamic in C#

## 🧩 Overview

Both `var` and `dynamic` are used to declare variables without explicitly specifying their data types.
However, they differ in how and when the type is determined.

## 🧠 Key Difference
| Feature                     | `var`                                                            | `dynamic`                                                                        |
| --------------------------- | ---------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| **Type Resolution**         | Compile-time (statically typed)                                  | Runtime (dynamically typed)                                                      |
| **Requires Initialization** | ✅ Yes, must be initialized at declaration                        | ❌ No, can be declared without initialization                                     |
| **Type Changes**            | ❌ Cannot change after initialization                             | ✅ Can change during runtime                                                      |
| **Performance**             | Faster (type checked at compile time)                            | Slower (type resolved at runtime)                                                |
| **IntelliSense Support**    | ✅ Full IntelliSense and compile-time checking                    | ⚠️ Limited IntelliSense, runtime errors possible                                 |
| **Use Case**                | When the type is known at compile time but you want cleaner code | When working with dynamic or unknown types (e.g., reflection, COM objects, JSON) |

## 🧾 Example
```csharp
// Using var - type known at compile time
var number = 10;         // number is int
number = "Hello";        // ❌ Compile-time error

// Using dynamic - type resolved at runtime
dynamic data = 10;       
data = "Hello";          // ✅ Valid (runtime type changes)
data = DateTime.Now;     // ✅ Also valid

```
## ⚠️ Important Notes

- `var` cannot be used as a method parameter or return type.

- `dynamic` can be used as a method parameter or return type.

- Errors in `var` are caught at `compile time`, while in `dynamic`, they appear at `runtime`.

# 🔧 9. Extension Methods in C#
## 🧩 Overview

`Extension methods` allow you to add new methods to existing types (classes, structs, or interfaces) without modifying their original source code or creating a new derived type.

They are a `static` method of a `static` class, but they are called as if they were instance methods on the extended type.

## 🧠 Key Points

- Declared as a `static` method inside a `static` class.

- The first parameter specifies which type the method extends and is prefixed with the `this` keyword.

- They enable you to add utility methods to existing .NET types or your own classes non-invasively.

## 🧾 Syntax
```csharp
public static class MyExtensions
{
    // Extension method for string type
    public static bool IsCapitalized(this string input)
    {
        if (string.IsNullOrEmpty(input))
            return false;

        return char.IsUpper(input[0]);
    }
}
class Program
{
    static void Main()
    {
        string word = "Hello";
        bool result = word.IsCapitalized(); // Calling extension method like instance method
        Console.WriteLine(result); // Output: True
    }
}

```
### 🧩 Behind the Scenes
Even though you call it like an `instance` method, the compiler translates it into a `static` method call.

```csharp
bool result = MyExtensions.IsCapitalized(word);
```
### ⚠️ Rules & Considerations

- The class containing extension methods must be `static`.

- The method itself must be  `static`.

- The first parameter defines which type is extended and must start with the keyword `this`.

- You cannot override existing methods using extension methods.

- **Namespace import (using)** is required to access extension methods defined elsewhere.

### 💡 Advantages

- Adds methods to existing types without inheritance or modifying source code.

- Improves code readability and reusability.

- Useful for **LINQ (Language Integrated Query)** , which heavily relies on extension methods.

### 🧩 Real-World Example: LINQ

LINQ methods like `Where()`, `Select()`, and `OrderBy()` are all extension methods on `IEnumerable<T>`.

```csharp
var numbers = new List<int> { 1, 2, 3, 4, 5 };
var evenNumbers = numbers.Where(n => n % 2 == 0); // Where is an extension method

```

### 🧾 Summary Table

| Feature             | Description                                              |
| ------------------- | -------------------------------------------------------- |
| **Definition**      | Adds new methods to existing types without altering them |
| **Class Type**      | Must be static                                           |
| **Method Type**     | Must be static                                           |
| **First Parameter** | Uses `this` keyword to specify the type being extended   |
| **Common Use**      | LINQ, utility/helper methods                             |
| **Cannot Do**       | Override existing methods                                |
