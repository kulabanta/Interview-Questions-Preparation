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

# 🔁 10. IEnumerable in C#

## 🧩 Overview

`IEnumerable` (and its generic version `IEnumerable<T>`) is an interface in C# that allows you to iterate over a collection of items one by one.
It’s the foundation of all collection types in .NET — arrays, lists, and other collections implement it.

## 🧠 Namespace & Definition

``` csharp
namespace System.Collections
{
    public interface IEnumerable
    {
        IEnumerator GetEnumerator();
    }
}

namespace System.Collections.Generic
{
    public interface IEnumerable<out T> : IEnumerable
    {
        IEnumerator<T> GetEnumerator();
    }
}
```
## ✅ Key Point:

- `IEnumerable<T>` returns an enumerator (`IEnumerator<T>`) that provides a way to iterate through the collection.

## ⚙️ Characteristics of IEnumerable
| Feature                | Description                                                    |
| ---------------------- | -------------------------------------------------------------- |
| **Namespace**          | `System.Collections` / `System.Collections.Generic`            |
| **Type of Execution**  | In-memory (client-side)                                        |
| **Execution Mode**     | Synchronous                                                    |
| **Supports LINQ?**     | ✅ Yes (LINQ to Objects)                                        |
| **Deferred Execution** | ✅ Yes (via LINQ methods)                                       |
| **Use Case**           | When working with in-memory collections like List, Array, etc. |

## 🧾 Basic Example
``` csharp
using System;
using System.Collections.Generic;

class Program
{
    static void Main()
    {
        IEnumerable<int> numbers = new List<int> { 1, 2, 3, 4, 5 };

        foreach (var num in numbers)
        {
            Console.WriteLine(num);
        }
    }
}

/**
    output
    1
    2
    3
    4
    5
**/
```
Here, `IEnumerable<int>` allows you to iterate over the collection of integers.

## 🧠 Custom IEnumerable Example
 You can also create your own class that implements `IEnumerable<T>`.

 ```csharp
using System;
using System.Collections;
using System.Collections.Generic;

public class FibonacciSequence : IEnumerable<int>
{
    private int _count;

    public FibonacciSequence(int count)
    {
        _count = count;
    }

    public IEnumerator<int> GetEnumerator()
    {
        int prev = 0, current = 1;
        for (int i = 0; i < _count; i++)
        {
            yield return prev;
            int temp = prev;
            prev = current;
            current = temp + current;
        }
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

class Program
{
    static void Main()
    {
        var fibonacci = new FibonacciSequence(5);

        foreach (var number in fibonacci)
        {
            Console.WriteLine(number);
        }
    }
}
/**
output
0
1
1
2
3
**/
 ```
 This demonstrates how you can implement custom iteration logic using `yield return`.

 ## 💡 Deferred Execution Example
 ```csharp
 IEnumerable<int> data = GetNumbers();

foreach (var num in data)
{
    Console.WriteLine(num);
}

IEnumerable<int> GetNumbers()
{
    Console.WriteLine("Start fetching numbers...");
    yield return 1;
    yield return 2;
    yield return 3;
    Console.WriteLine("Done fetching numbers!");
}

/**
Start fetching numbers...
1
2
3
Done fetching numbers!
**/
 ```

## ⚖️ Advantages
✅ Simple way to iterate over collections<br>
✅ Supports LINQ and deferred execution <br>
✅ Reduces memory usage (fetches data lazily)<br>
✅ Enables custom iteration logic using `yield`

## ⚠️ Limitations

❌ Read-only access (cannot modify elements)<br>
❌ No indexing (unlike IList)<br>
❌ Single-pass enumeration (must re-enumerate to access again)<br>
❌ Works in-memory only (unlike `IQueryable`)<br>

## 🧩 Interview Tip
```
Q: What is the difference between IEnumerable and IEnumerator?
A:
- IEnumerable provides an enumerator using GetEnumerator()
- IEnumerator actually performs the iteration using MoveNext() and Current.
```

# 🔁 11. yield Keyword in C#

## 🧩 Overview

- The `yield` keyword in C# is used to simplify the creation of iterators (methods that return elements one at a time).<br>
- It allows you to return items lazily, one by one, without creating an intermediate collection or managing the iteration state manually.
- When a method uses `yield` return, the compiler automatically generates the logic for `IEnumerable` or `IEnumerator`.

## ⚙️ Syntax
```csharp
yield return <expression>;
yield break;
```

- yield return — returns the next element in the sequence.

- yield break — stops the iteration early.

## 🧠 Key Points
| Feature          | Description                                                                     |
| ---------------- | ------------------------------------------------------------------------------- |
| **Purpose**      | Used to create iterator methods easily                                          |
| **Return Type**  | Must return `IEnumerable`, `IEnumerable<T>`, `IEnumerator`, or `IEnumerator<T>` |
| **Execution**    | Deferred (lazy evaluation)                                                      |
| **Keyword Type** | Contextual keyword                                                              |
| **Control Flow** | Saves the current state of iteration between calls                              |

## 🧾 Example 1: Basic Use of yield return
```csharp
using System;
using System.Collections.Generic;

class Program
{
    static void Main()
    {
        foreach (int num in GetNumbers())
        {
            Console.WriteLine(num);
        }
    }

    static IEnumerable<int> GetNumbers()
    {
        yield return 1;
        yield return 2;
        yield return 3;
    }
}
/**
1
2
3
**/
```
- 💡 The compiler automatically creates an iterator object that remembers where it left off after each `yield` return.

## 🧩 Example 2: Using yield break
```csharp
static IEnumerable<int> GetNumbers()
{
    for (int i = 1; i <= 5; i++)
    {
        if (i == 4)
            yield break; // stop iteration

        yield return i;
    }
}
/**
1
2
3
**/
```
- Here, iteration stops when `i == 4` due to `yield break`.

## ⚡ Example 3: Deferred Execution
`yield` supports deferred execution — elements are generated only when iterated.
```csharp
static IEnumerable<int> GetNumbers()
{
    Console.WriteLine("Start");
    yield return 1;
    yield return 2;
    Console.WriteLine("End");
}

static void Main()
{
    var numbers = GetNumbers();
    Console.WriteLine("Before iteration");

    foreach (var n in numbers)
        Console.WriteLine(n);
}
/**
Before iteration
Start
1
2
End
**/
```
💡 The method executes only during iteration, not when `GetNumbers()` is called — demonstrating lazy evaluation.

## 🧾 When to Use yield

✅ Use yield when:

1. You want to stream data one item at a time.

2. You need deferred execution.

3. You want to simplify iterators without creating temporary collections.

4. You are implementing custom `IEnumerable` logic.

## ⚠️ Limitations of yield

❌ Cannot use inside:

1. async methods (use `IAsyncEnumerable` instead).

2. Anonymous methods or lambda expressions (directly).

3. Methods that return anything other than IEnumerable/IEnumerator.

## 💡 Interview Tip
```
Q: What is the advantage of using yield in C#?
A: It simplifies iterator creation, enables deferred execution, and reduces memory usage by returning data lazily instead of building large collections in memory.
```

# 🔄 12. IEnumerator in C#

## 🧩 Overview

`IEnumerator` in C# is an interface that allows sequential iteration over a collection (like arrays, lists, etc.).<br>
It provides the mechanism to traverse items one at a time without exposing the underlying collection structure.

## 🧠 Namespace & Definition
```csharp
namespace System.Collections
{
    public interface IEnumerator
    {
        bool MoveNext();     // Moves to the next element
        object Current { get; }  // Gets the current element
        void Reset();        // Resets the enumerator to its initial position
    }
}

namespace System.Collections.Generic
{
    public interface IEnumerator<out T> : IDisposable, IEnumerator
    {
        new T Current { get; } // Type-safe version of Current
    }
}

```
## ⚙️ Key Members
| Member           | Description                                                                          |
| ---------------- | ------------------------------------------------------------------------------------ |
| **`Current`**    | Returns the current element in the collection.                                       |
| **`MoveNext()`** | Moves the cursor to the next element; returns `false` if there are no more elements. |
| **`Reset()`**    | Resets the enumerator to its initial position (before the first element).            |
| **`Dispose()`**  | Releases unmanaged resources (in the generic version).                               |

## 🧾 Example: Using IEnumerator Manually
```csharp
using System;
using System.Collections;

class Program
{
    static void Main()
    {
        ArrayList numbers = new ArrayList { 1, 2, 3 };

        IEnumerator enumerator = numbers.GetEnumerator();

        while (enumerator.MoveNext())
        {
            Console.WriteLine(enumerator.Current);
        }
    }
}
/**
1
2
3
**/

```
💡 The foreach loop in C# internally uses IEnumerator to iterate over a collection.

## 🧠 How foreach Uses IEnumerator
```csharp
foreach (var item in collection)
{
    // Behind the scenes:
    // 1. Calls GetEnumerator()
    // 2. Calls MoveNext()
    // 3. Accesses Current
}

```
Equivalent to:
```cshap
var enumerator = collection.GetEnumerator();
try
{
    while (enumerator.MoveNext())
    {
        var item = enumerator.Current;
        Console.WriteLine(item);
    }
}
finally
{
    (enumerator as IDisposable)?.Dispose();
}

```

## 🔁 Relationship Between IEnumerable and IEnumerator
| Concept         | Description                                                         |
| --------------- | ------------------------------------------------------------------- |
| **IEnumerable** | Provides an enumerator using `GetEnumerator()`                      |
| **IEnumerator** | Provides the actual mechanism to iterate (MoveNext, Current, Reset) |
| **Link**        | `IEnumerable.GetEnumerator()` returns an `IEnumerator`              |

## Visual Flow
```csharp
foreach → calls GetEnumerator() → returns IEnumerator
→ MoveNext() → Current → repeat until MoveNext() = false

```
## ⚖️ Advantages

✅ Enables custom iteration logic<br>
✅ Separates collection logic from traversal logic<br>
✅ Works with foreach loop<br>
✅ Reduces memory overhead (no copying of collection)<br>

## ⚠️ Limitations

❌ One-way traversal (no backward iteration)<br>
❌ State is lost after iteration<br>
❌ Not thread-safe<br>
❌ Must call `Reset()` to start over (usually avoided)<br>

## 🧾 Summary Table
| Feature            | Description                                         |
| ------------------ | --------------------------------------------------- |
| **Interface Name** | `IEnumerator` / `IEnumerator<T>`                    |
| **Namespace**      | `System.Collections` / `System.Collections.Generic` |
| **Key Methods**    | `MoveNext()`, `Current`, `Reset()`                  |
| **Purpose**        | Iterates sequentially through a collection          |
| **Common Usage**   | Used internally by `foreach`                        |
| **Simplified By**  | `yield` keyword                                     |

# ⚙️ 13. IQueryable in C#

## 🧩 Overview

`IQueryable` is an interface in C# that allows querying data from a data source (like a database) in a deferred and composable way.<br>
It is part of the `System.Linq` namespace and is commonly used in Entity Framework and LINQ to SQL for querying data sources that support query translation.

## 🧠 Definition
```csharp
public interface IQueryable : IEnumerable
{
    Type ElementType { get; }
    Expression Expression { get; }
    IQueryProvider Provider { get; }
}

```
`IQueryable<T>` inherits from `IEnumerable<T>` and adds the ability to build expression trees that represent queries, which are later executed by a query provider (e.g., EF Core).

## 🔍 Key Characteristics
| Feature            | Description                                                                  |
| ------------------ | ---------------------------------------------------------------------------- |
| **Namespace**      | `System.Linq`                                                                |
| **Inherits From**  | `IEnumerable<T>`                                                             |
| **Execution Type** | Deferred (query executed when enumerated)                                    |
| **Query Provider** | Uses `IQueryProvider` to translate queries into SQL or other query languages |
| **Use Case**       | When querying remote data sources (like databases, APIs)                     |

## 💡 Example — Basic Usage
```csharp
using System;
using System.Linq;
using System.Collections.Generic;

public class Program
{
    public static void Main()
    {
        List<int> numbers = new List<int> { 1, 2, 3, 4, 5 };
        
        // IQueryable example (simulating LINQ to SQL or EF)
        IQueryable<int> queryableNumbers = numbers.AsQueryable();

        var result = queryableNumbers
                        .Where(n => n > 2)
                        .Select(n => n * n);

        foreach (var num in result)
        {
            Console.WriteLine(num);
        }
    }
}
/**
9
16
25
**/
```
## ⚙️ How It Works

1. **Expression Tree Creation:**<br>
i. When you write a query like .Where(n => n > 2), it is not executed immediately.<br>
ii. Instead, an expression tree is built, representing the query.

2. **Deferred Execution:**<br>
The query executes only when enumerated (e.g., in a `foreach` loop or .`ToList()` call).

3. **Query Translation:**<br>
In ORMs like Entity Framework, the expression tree is translated to SQL and executed in the database.

## 🧾 Example — Entity Framework Usage
```csharp
using (var context = new AppDbContext())
{
    IQueryable<User> usersQuery = context.Users
                                         .Where(u => u.Age > 25);

    // Query not executed yet
    var userList = usersQuery.ToList(); // Query executed here
}

```
## ✅ Explanation:

- The `Where()` builds a query expression.
- The actual SQL query is executed only when `.ToList()` is called.

## ⚖️ Difference Between IEnumerable and IQueryable
| Feature                | `IEnumerable`                          | `IQueryable`                |
| ---------------------- | -------------------------------------- | --------------------------- |
| **Execution**          | In-memory                              | Database/server-side        |
| **Query Translation**  | LINQ to Objects                        | LINQ to SQL / EF            |
| **Performance**        | Fetches all data and filters in memory | Fetches only filtered data  |
| **Deferred Execution** | Yes                                    | Yes                         |
| **Best For**           | Small in-memory collections            | Large external data sources |

# ⚙️ 14. IAsyncEnumerable<T> in C#
## 🧩 Overview

`IAsyncEnumerable<T>` is an interface introduced in C# 8.0 that allows you to iterate over a sequence of data asynchronously using the await foreach loop. <br>

It’s useful when working with data streams or asynchronous operations — especially when retrieving large datasets, reading files, or making database/network calls where waiting synchronously would block the thread.

## 🧠 Definition
```csharp
public interface IAsyncEnumerable<out T>
{
    IAsyncEnumerator<T> GetAsyncEnumerator(CancellationToken cancellationToken = default);
}

```
## 🔍 Key Characteristics
| Feature             | Description                                       |
| ------------------- | ------------------------------------------------- |
| **Namespace**       | `System.Collections.Generic`                      |
| **Introduced In**   | C# 8.0 / .NET Core 3.0+                           |
| **Purpose**         | To enable asynchronous iteration over collections |
| **Execution**       | Asynchronous (non-blocking)                       |
| **Enumerator Type** | `IAsyncEnumerator<T>`                             |
| **Loop Type**       | `await foreach`                                   |

## 💡 Example — Basic Usage
```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

public class Program
{
    public static async IAsyncEnumerable<int> GenerateNumbersAsync()
    {
        for (int i = 1; i <= 5; i++)
        {
            await Task.Delay(500); // Simulate async operation
            yield return i;
        }
    }

    public static async Task Main()
    {
        await foreach (var number in GenerateNumbersAsync())
        {
            Console.WriteLine(number);
        }
    }
}

/**
1
2
3
4
5
**/
```
## ✅ Explanation:

1. The method `GenerateNumbersAsync()` returns `IAsyncEnumerable<int>`.

2. Each iteration awaits the completion of `Task.Delay(500)`.

3. `await foreach` consumes the asynchronous sequence.

## 🧩 Example — Database Scenario (EF Core)
```csharp
using var context = new AppDbContext();

await foreach (var user in context.Users.AsAsyncEnumerable())
{
    Console.WriteLine(user.Name);
}

```
## ✅ Explanation:

1. `AsAsyncEnumerable()` converts a database query to an asynchronous stream.

2. The `await foreach` loop asynchronously reads rows as they’re available, improving scalability for large result sets.

## ⚖️ Difference Between `IEnumerable<T>` and `IAsyncEnumerable<T>`
| Feature             | `IEnumerable<T>`            | `IAsyncEnumerable<T>`           |
| ------------------- | --------------------------- | ------------------------------- |
| **Iteration Type**  | Synchronous                 | Asynchronous                    |
| **Introduced In**   | C# 1.0                      | C# 8.0                          |
| **Enumerator**      | `IEnumerator<T>`            | `IAsyncEnumerator<T>`           |
| **Loop**            | `foreach`                   | `await foreach`                 |
| **Use Case**        | In-memory, fast collections | I/O-bound or network operations |
| **Thread Blocking** | Blocks thread               | Non-blocking, async             |


## ⚠️ Important Notes
1. You must mark the consuming method as `async` when using `await foreach`.

2. The source method returning `IAsyncEnumerable<T>` can use `yield return` with `await` for asynchronous data generation.

3. Works best with I/O-bound operations (database, APIs, streams, etc.).


# ⚙️ 15. Generics in C#

## 🧩 Overview

**Generics** in C# allow you to define **type-safe** data structures and methods without committing to a specific data type until the class, method, or interface is instantiated or called.

They help in code reusability, type safety, and performance improvement by avoiding boxing/unboxing and runtime type casting.

## 🧠 Definition
```csharp
// Generic class
public class GenericClass<T>
{
    public T Data { get; set; }

    public void Display(T value)
    {
        Console.WriteLine($"Value: {value}");
    }
}
public class Program
{
    public static void Main()
    {
        // Integer type
        GenericClass<int> intObj = new GenericClass<int>();
        intObj.Display(100);

        // String type
        GenericClass<string> strObj = new GenericClass<string>();
        strObj.Display("Hello Generics");
    }
}
/**
Value: 100
Value: Hello Generics

**/
```

## 🔍 Why Use Generics?
| Feature              | Description                                 |
| -------------------- | ------------------------------------------- |
| **Type Safety**      | Ensures compile-time checking of types      |
| **Code Reusability** | Write once, use for any type                |
| **Performance**      | Avoids boxing/unboxing and casting overhead |
| **Maintainability**  | Reduces code duplication and runtime errors |

## 🧩 Generic Methods
You can also create **generic methods** inside **normal or generic classes**.
```csharp
public class Program
{
    public static void Main()
    {
        // Integer type
        GenericClass<int> intObj = new GenericClass<int>();
        intObj.Display(100);

        // String type
        GenericClass<string> strObj = new GenericClass<string>();
        strObj.Display("Hello Generics");
    }
}

```
## 🧰 Generic Constraints
You can **restrict the type parameters** using constraints for more control.

| Constraint                | Description                                  |
| ------------------------- | -------------------------------------------- |
| `where T : struct`        | Only value types                             |
| `where T : class`         | Only reference types                         |
| `where T : new()`         | Must have a public parameterless constructor |
| `where T : BaseClass`     | Must inherit from a specific base class      |
| `where T : InterfaceName` | Must implement a specific interface          |

```csharp
public class Repository<T> where T : class, new()
{
    public T CreateInstance()
    {
        return new T(); // Works because of new() constraint
    }
}

```

## 🧾 Example — Generic Interface
```csharp
public interface IRepository<T>
{
    void Add(T item);
    IEnumerable<T> GetAll();
}

public class Repository<T> : IRepository<T>
{
    private List<T> _items = new List<T>();

    public void Add(T item) => _items.Add(item);
    public IEnumerable<T> GetAll() => _items;
}

```
## 🧠 Advanced Concept — Generic Delegates
C# also supports **generic delegates**:
```csharp
Func<int, string> convert = num => $"Number: {num}";
Action<string> print = msg => Console.WriteLine(msg);
Predicate<int> isEven = num => num % 2 == 0;
```

## ✅ Interview Tip
**Q**. Why are generics better than using object type parameters?<br>
**Answer:**<br>
Because generics ensure compile-time type safety, avoid runtime casting, and improve performance by preventing boxing/unboxing.

# ⚙️ 16. Delegates in C#
## 🧩 Overview

A **delegate** in C# is a type-safe function pointer — it holds a reference to a method and allows methods to be passed as parameters, invoked dynamically, or used as callbacks.

Delegates enable decoupling between the caller and the method being called, supporting event-driven and functional programming in C#.

## 🧠 Definition
```csharp
// Delegate Declaration
public delegate void MyDelegate(string message);
```

## ✅ Explanation:

- delegate keyword defines a delegate type.

- The delegate can hold a reference to any method with a matching signature and return type.

## 💡 Example — Basic Delegate Usage
```csharp
using System;

public class Program
{
    // Step 1: Declare a delegate
    public delegate void GreetDelegate(string name);

    // Step 2: Create methods that match the delegate signature
    public static void GreetMorning(string name) => Console.WriteLine($"Good morning, {name}!");
    public static void GreetEvening(string name) => Console.WriteLine($"Good evening, {name}!");

    public static void Main()
    {
        // Step 3: Instantiate delegate
        GreetDelegate greet = GreetMorning;

        // Step 4: Invoke delegate
        greet("Alice");

        // Step 5: Reassign or combine methods
        greet += GreetEvening;
        greet("Bob");
    }
}
/**
Good morning, Alice!
Good morning, Bob!
Good evening, Bob!
**/
```

## ✅ Explanation:

- The delegate points to a method and can invoke it.
- You can combine multiple methods using += (multicast delegate).

## 🧰 Why Use Delegates?
| Benefit                    | Description                                           |
| -------------------------- | ----------------------------------------------------- |
| **Decoupling**             | Caller doesn’t need to know which method will execute |
| **Callback Methods**       | Enables passing methods as parameters                 |
| **Multicasting**           | Can invoke multiple methods at once                   |
| **Event Handling**         | Core mechanism for events in C#                       |
| **Functional Programming** | Supports lambdas and anonymous functions              |

## 🧩 Types of Delegates
1. **Single-cast Delegate**

- Holds a reference to one method.

```csharp
Action<string> greet = name => Console.WriteLine($"Hello, {name}");
greet("Alice");
```

2. **Multicast Delegate**
- Can hold multiple methods and invoke them in sequence.
```csharp
MyDelegate greetAll = GreetMorning;
greetAll += GreetEvening;
greetAll("John");
```
3. **Generic Delegates**
- C# provides built-in generic delegate types:

| Delegate Type      | Description                                    | Example                                          |
| ------------------ | ---------------------------------------------- | ------------------------------------------------ |
| `Action<T>`        | Represents a method with **no return value**   | `Action<int> print = x => Console.WriteLine(x);` |
| `Func<T, TResult>` | Represents a method **that returns a value**   | `Func<int, int> square = x => x * x;`            |
| `Predicate<T>`     | Represents a method that **returns a boolean** | `Predicate<int> isEven = x => x % 2 == 0;`       |

## ⚙️ Delegate with Anonymous Method
```csharp
MyDelegate showMessage = delegate(string msg)
{
    Console.WriteLine($"Message: {msg}");
};

showMessage("Hello from anonymous method!");
```

## 🧠 Delegate with Lambda Expression
```csharp
MyDelegate display = (msg) => Console.WriteLine($"Lambda says: {msg}");
display("Hello from Lambda!");
```

## 🧾 Example — Delegate as a Callback
```csharp
public class MathOperations
{
    public delegate void ResultDelegate(int result);

    public void Add(int a, int b, ResultDelegate callback)
    {
        int sum = a + b;
        callback(sum); // Delegate callback
    }
}

public class Program
{
    public static void Main()
    {
        MathOperations math = new MathOperations();
        math.Add(5, 10, result => Console.WriteLine($"Sum: {result}"));
    }
}
```
## ⚖️ Difference Between Delegate, Func, Action, and Predicate
| Type               | Return Type    | Parameters     | Description                            |
| ------------------ | -------------- | -------------- | -------------------------------------- |
| `Delegate`         | Custom defined | Custom defined | Base concept                           |
| `Action<T>`        | `void`         | 0–16           | Built-in delegate with no return value |
| `Func<T, TResult>` | Any            | 0–16           | Built-in delegate that returns a value |
| `Predicate<T>`     | `bool`         | 1              | Used for conditions and filtering      |

## 🧠 Interview Tip
**Q.** What’s the difference between Delegates and Events?<br>
**Answer:** <br>
- A delegate is a reference to a method.
- An event is a wrapper around a delegate that restricts access — only the declaring class can invoke it, while others can subscribe/unsubscribe.

# ⚙️17. Events in C#

## 🧩 Overview

An event in C# is a mechanism for communication between objects.
It allows a class (publisher) to notify other classes (subscribers) when something happens.

Events are built on top of delegates — they provide a safe and structured way to expose delegate functionality, where:

- Only the publisher can raise (trigger) the event.

- Other classes can **subscribe (+=)** or **unsubscribe (-=)** to be notified.

## 💡 Example — Basic Event Usage
```csharp
using System;

public class Process
{
    // Step 1: Declare delegate
    public delegate void Notify();

    // Step 2: Declare event
    public event Notify ProcessCompleted;

    public void StartProcess()
    {
        Console.WriteLine("Process started...");
        System.Threading.Thread.Sleep(1000); // Simulate work
        Console.WriteLine("Process finished!");

        // Step 3: Raise event
        ProcessCompleted?.Invoke();
    }
}

public class Program
{
    public static void Main()
    {
        Process process = new Process();

        // Step 4: Subscribe to event
        process.ProcessCompleted += OnProcessCompleted;

        process.StartProcess();
    }

    // Step 5: Event handler
    static void OnProcessCompleted()
    {
        Console.WriteLine("Notification: Process Completed Successfully!");
    }
}
/**
output 
Process started...
Process finished!
Notification: Process Completed Successfully!
**/
```
### ✅ Explanation:

1. `event` keyword declares an event based on a delegate type.

2. `?.Invoke()` ensures the event is raised only if there are subscribers.

## 🧰 Why Use Events?
| Benefit             | Description                                                            |
| ------------------- | ---------------------------------------------------------------------- |
| **Loose Coupling**  | Subscribers don’t need to know the internal logic of the publisher     |
| **Reusability**     | Multiple subscribers can listen to one event                           |
| **Encapsulation**   | Only the declaring class can raise the event                           |
| **Maintainability** | Clear separation between trigger (publisher) and response (subscriber) |

## 🧩 Event with EventHandler
C# provides the `EventHandler` delegate as a built-in standard for defining events.
```csharp
public class Process
{
    // Standard event pattern using EventHandler
    public event EventHandler ProcessCompleted;

    public void StartProcess()
    {
        Console.WriteLine("Process running...");
        OnProcessCompleted();
    }

    protected virtual void OnProcessCompleted()
    {
        ProcessCompleted?.Invoke(this, EventArgs.Empty);
    }
}

public class Program
{
    static void Main()
    {
        Process process = new Process();
        process.ProcessCompleted += ProcessCompletedHandler;
        process.StartProcess();
    }

    static void ProcessCompletedHandler(object sender, EventArgs e)
    {
        Console.WriteLine("Process completed event received!");
    }
}
```
### ✅ Explanation:
- Uses the standard EventHandler delegate:

```csharp
public delegate void EventHandler(object sender, EventArgs e);
```

- sender → the object that raised the event.

- EventArgs → carries additional event data (empty here).

## ⚙️ Event with Custom EventArgs
- You can send custom data through events using a class derived from `EventArgs`.
```csharp
public class ProcessEventArgs : EventArgs
{
    public string Message { get; set; }
}

public class Process
{
    public event EventHandler<ProcessEventArgs> ProcessCompleted;

    public void StartProcess()
    {
        Console.WriteLine("Processing...");
        OnProcessCompleted(new ProcessEventArgs { Message = "Task done successfully!" });
    }

    protected virtual void OnProcessCompleted(ProcessEventArgs e)
    {
        ProcessCompleted?.Invoke(this, e);
    }
}

public class Program
{
    static void Main()
    {
        Process process = new Process();
        process.ProcessCompleted += (sender, e) =>
        {
            Console.WriteLine($"Event received: {e.Message}");
        };

        process.StartProcess();
    }
}
```
### ✅ Explanation:

1. `ProcessEventArgs` holds custom event data.

2. `EventHandler<TEventArgs>` is a generic delegate that passes additional info.

## ⚖️ Difference Between Delegate and Event

| Feature            | Delegate                            | Event                                             |
| ------------------ | ----------------------------------- | ------------------------------------------------- |
| **Definition**     | Type-safe function pointer          | Wrapper around a delegate                         |
| **Invocation**     | Can be called directly by any class | Can only be invoked by the class that declared it |
| **Access Control** | Publicly accessible                 | Restricted to publisher                           |
| **Use Case**       | Callbacks, anonymous methods        | Notifications, message broadcasting               |


### 🧠 Interview Tip
**Q.** What’s the difference between an Event and a Delegate?<br>
**Answer**: A delegate is a method reference type, while an event is a language feature that restricts access to delegates so that only the declaring class can invoke them. Events are the publish-subscribe layer built on top of delegates.

# ⚙️18. Task in C#
## 🧩 Definition

A `Task` in C# represents an `asynchronous operation that can run independently of the main thread`.
It is part of the `Task Parallel Library (TPL)` and is defined in the `System.Threading.Tasks` namespace.

Tasks simplify concurrent programming by handling thread management, synchronization, and result returning automatically.

## 🧠 Purpose

- To run code asynchronously without blocking the main thread.

- To return results from background operations.

- To handle exceptions, cancellation, and continuations easily.

- To improve performance and responsiveness in applications.

## 🧾 Basic Example
```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static void Main()
    {
        Task task = Task.Run(() =>
        {
            for (int i = 1; i <= 5; i++)
            {
                Console.WriteLine($"Task running: {i}");
            }
        });

        task.Wait(); // Wait for the task to complete
        Console.WriteLine("Main method completed.");
    }
}
/**
Task running: 1
Task running: 2
Task running: 3
Task running: 4
Task running: 5
Main method completed.
**/
```

### 🔹 Task with Return Value
```csharp
Task<int> task = Task.Run(() =>
{
    int sum = 0;
    for (int i = 1; i <= 5; i++)
        sum += i;
    return sum;
});

int result = task.Result; // Blocks until completed
Console.WriteLine($"Sum: {result}");

/**
Sum: 15
**/
```
### 🔹 Task with async/await (Recommended Way)
```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        int result = await CalculateSumAsync();
        Console.WriteLine($"Result: {result}");
    }

    static async Task<int> CalculateSumAsync()
    {
        await Task.Delay(1000); // Simulate work
        return 5 + 10;
    }
}
/**
Result: 15
**/
```
## ⚙️ Key Features of Task
| Feature                  | Description                                  |
| ------------------------ | -------------------------------------------- |
| **Namespace**            | `System.Threading.Tasks`                     |
| **Return Type**          | `Task` or `Task<TResult>`                    |
| **Supports async/await** | Yes ✅                                        |
| **Cancellation**         | Via `CancellationToken`                      |
| **Exception Handling**   | Through `try/catch` or `task.ContinueWith()` |
| **Thread Pool Usage**    | Uses threads from the CLR-managed pool       |
| **Continuation**         | Allows chaining tasks using `ContinueWith()` |

## 🧩 Example: Task Continuation
```csharp
Task.Run(() => Console.WriteLine("Task 1"))
    .ContinueWith(t => Console.WriteLine("Task 2 after Task 1"))
    .ContinueWith(t => Console.WriteLine("Task 3 after Task 2"));
/**
Task 1
Task 2 after Task 1
Task 3 after Task 2
**/
```

### 🔹 Task Cancellation Example
```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        CancellationTokenSource cts = new();
        var token = cts.Token;

        Task task = Task.Run(() =>
        {
            for (int i = 0; i < 10; i++)
            {
                token.ThrowIfCancellationRequested();
                Console.WriteLine($"Working... {i}");
                Thread.Sleep(300);
            }
        }, token);

        cts.CancelAfter(1000); // Cancel after 1 second

        try
        {
            await task;
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("Task was cancelled.");
        }
    }
}
/**
Working... 0  
Working... 1  
Working... 2  
Task was cancelled.

**/
```
## 🆚 Task vs Thread (Quick Comparison)
| Feature                  | **Task**                | **Thread**                   |
| ------------------------ | ----------------------- | ---------------------------- |
| **Abstraction Level**    | High                    | Low                          |
| **Return Value**         | ✅ Yes (`Task<TResult>`) | ❌ No                         |
| **Cancellation Support** | ✅ Yes                   | ❌ No                         |
| **Exception Handling**   | Easy (`try/catch`)      | Manual                       |
| **Async/Await Support**  | ✅ Yes                   | ❌ No                         |
| **Performance**          | Efficient (Thread Pool) | Expensive (dedicated thread) |

## 💡 Interview Tip
✅ “A Task in C# represents an asynchronous operation.
It’s a higher-level abstraction built on top of threads, offering better performance, exception handling, and async support.”

# ⚙️19. Thread in C#
## 🧩 Definition

A `Thread` in C# represents the `smallest unit of execution within a process`.
Each thread can execute code independently, allowing multiple operations to run concurrently.

Threads are managed by the operating system and are part of the `System.Threading` namespace.

## 🧠 Purpose
- To perform multiple tasks simultaneously (parallelism).

- To improve responsiveness in applications (e.g., UI remains active while background work runs).

- To run background operations independently of the main program flow.

## 🧾 Basic Example: Creating and Starting a Thread
```csharp
using System;
using System.Threading;

class Program
{
    static void Main()
    {
        Thread thread = new Thread(PrintNumbers);
        thread.Start(); // Start the new thread

        Console.WriteLine("Main thread continues...");
    }

    static void PrintNumbers()
    {
        for (int i = 1; i <= 5; i++)
        {
            Console.WriteLine($"Worker thread: {i}");
            Thread.Sleep(500); // Simulate some work
        }
    }
}
/**
Main thread continues...
Worker thread: 1
Worker thread: 2
Worker thread: 3
Worker thread: 4
Worker thread: 5
**/
```
### 🔹 Creating a Thread with Lambda Expression
```csharp
Thread thread = new Thread(() =>
{
    Console.WriteLine("Thread started using lambda!");
});
thread.Start();
```

## ⚙️ Key Properties of Thread
| Property      | Description                                          |
| ------------- | ---------------------------------------------------- |
| **Namespace** | `System.Threading`                                   |
| **Start()**   | Begins execution of the thread                       |
| **Sleep(ms)** | Pauses execution for a specified time                |
| **Abort()**   | Terminates the thread (⚠️ Deprecated)                |
| **IsAlive**   | Checks if the thread is still running                |
| **Join()**    | Blocks the calling thread until the thread completes |

## 🧩 Example: Using Join()
```csharp
Thread thread = new Thread(() =>
{
    Console.WriteLine("Thread starting...");
    Thread.Sleep(2000);
    Console.WriteLine("Thread completed.");
});

thread.Start();
thread.Join(); // Wait for the thread to finish before continuing
Console.WriteLine("Main thread resumes after join.");

/**
Thread starting...
Thread completed.
Main thread resumes after join.
**/
```
### 🔹 Foreground vs Background Threads
| Type                  | Description                                            |
| --------------------- | ------------------------------------------------------ |
| **Foreground Thread** | Keeps the process alive until it finishes (default).   |
| **Background Thread** | Automatically ends when all foreground threads finish. |

```csharp
Thread backgroundThread = new Thread(() =>
{
    Console.WriteLine("Background thread running...");
    Thread.Sleep(2000);
    Console.WriteLine("Background thread completed.");
});

backgroundThread.IsBackground = true;
backgroundThread.Start();
Console.WriteLine("Main thread ending...");
/**
Background thread running...
Main thread ending...
**/
// (The program may end before the background thread finishes.)
```
## 🧩 Example: Thread with Parameters
```csharp
Thread thread = new Thread(PrintMessage);
thread.Start("Hello from Thread!");

static void PrintMessage(object message)
{
    Console.WriteLine(message);
}
// Hello from Thread!
```
## ⚙️ Thread Safety
When multiple threads access shared data, it can cause `race conditions` or `data corruption`.
To prevent this, use `locks` or `thread synchronization` techniques.

### Example using `lock` keyword:
```csharp
static object lockObj = new();
static int counter = 0;

static void Increment()
{
    for (int i = 0; i < 1000; i++)
    {
        lock (lockObj)
        {
            counter++;
        }
    }
}

```
## 🧠 Advantages of Using Threads

- Improves application responsiveness.

- Enables parallel execution of code.

- Utilizes multiple CPU cores efficiently.

## ⚠️ Disadvantages
- Difficult to manage and debug.

- Can cause race conditions and deadlocks.

- Consumes more memory and resources than higher-level abstractions like Task.

## 💡 Interview Tip
✅ “A Thread is the smallest unit of execution in a process.
It allows concurrent code execution but requires manual management of synchronization and lifecycle.”

# 📘 20. Records in C#
## 🧩 What Are Records?
Records are `reference types` introduced in C# 9.0 designed to make it easier to create `immutable data models (data-carrying objects)` with built-in value equality.

**A record is like a class, but it’s optimized for holding data rather than behavior.**

## 🧱 Key Characteristics
| Feature                       | Description                                                                                                                                  |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Immutability**              | Record properties are typically *init-only* — they can be set only during initialization.                                                    |
| **Value-based Equality**      | Two records with the same data are considered *equal*, unlike classes which use *reference equality* by default.                             |
| **Concise Syntax**            | Records can be declared with a *positional syntax* that automatically generates constructors, `Equals()`, `GetHashCode()`, and `ToString()`. |
| **Built-in with-expressions** | You can easily create copies of records with modified properties using the `with` expression.                                                |

## 🧠 Syntax
### 1. Basic Record Declaration
```csharp
public record Person
{
    public string FirstName { get; init; }
    public string LastName { get; init; }
}
```
### 2. Positional Record (Concise Form)
```csharp
public record Person(string FirstName, string LastName);
```
This automatically generates:

- A constructor with parameters FirstName and LastName

- Read-only properties

- Equals(), GetHashCode(), and ToString()

- A `Deconstruct()` method

## 🔁 Example: Equality
```csharp
var p1 = new Person("John", "Doe");
var p2 = new Person("John", "Doe");

Console.WriteLine(p1 == p2); // ✅ True (Value equality)
```
_If this were a class, the output would be `False` because classes compare `references by default`._

## 🧍‍♂️ Copying with `with` Expression
You can create a copy of a record while changing some values:
```csharp
var p1 = new Person("John", "Doe");
var p2 = p1 with { LastName = "Smith" };

Console.WriteLine(p2); // Output: Person { FirstName = John, LastName = Smith }
```

## 🧩 Inheritance with Records
Records can be `inherited` — and they preserve `value equality` across hierarchies.

```csharp
public record Person(string Name);
public record Employee(string Name, string Department) : Person(Name);
```

## 🧍‍♀️ Mutable Records (Not Recommended)
You can make record properties mutable, but that goes against their design for immutability.
```csharp
public record MutablePerson
{
    public string Name { get; set; } // mutable
}
```
## ⚖️ Record vs Class
| Feature          | Record                          | Class                    |
| ---------------- | ------------------------------- | ------------------------ |
| Equality         | Value-based                     | Reference-based          |
| Immutability     | Default (with `init`)           | Mutable by default       |
| Boilerplate Code | Minimal                         | Verbose                  |
| Typical Use Case | DTOs, Models, Immutable Objects | Business Logic, Entities |

## Accessing Properties
You can access record properties just like class properties — using the `dot (.)` operator.
```csharp
Console.WriteLine(person.FirstName);  // Output: John
Console.WriteLine(person.LastName);   // Output: Doe
```
### ✅ Explanation:

- Each property in a record is `public` and `read-only` by default (when using positional syntax).

- You can read its value directly.

## Records with Property Initializers
If you define a record using property syntax, you can access the properties the same way:
```csharp
public record Employee
{
    public string Name { get; init; }
    public string Department { get; init; }
}

var emp = new Employee { Name = "Alice", Department = "HR" };

Console.WriteLine(emp.Name);        // Output: Alice
Console.WriteLine(emp.Department);  // Output: HR
```

## Using Deconstruction (Optional Way)
You can `deconstruct` record properties into local variables.
```csharp
var person = new Person("John", "Doe");

var (firstName, lastName) = person;

Console.WriteLine(firstName); // Output: John
Console.WriteLine(lastName);  // Output: Doe
```

## 🧭 Summary
| Concept           | Description                                    |
| ----------------- | ---------------------------------------------- |
| **Record Type**   | Reference type optimized for immutable data    |
| **Introduced In** | C# 9.0                                         |
| **Equality Type** | Value-based                                    |
| **Common Usage**  | DTOs, Immutable Models, Data Transfer          |
| **Key Feature**   | `with` expression for non-destructive mutation |

# 🧩 21. Tuples in C#

## What is a Tuple?
`Tuple` in C# is a data structure that allows you to store `multiple values of different data types` in a `single object without creating a custom class or struct`.

It’s often used to `return multiple values from a method` easily.

## Syntax
✅ Using Classic Tuple Class (Before C# 7.0)
```csharp
Tuple<int, string, bool> person = new Tuple<int, string, bool>(1, "John", true);

Console.WriteLine(person.Item1); // 1
Console.WriteLine(person.Item2); // John
Console.WriteLine(person.Item3); // True
```
🔸 Item1, Item2, etc., are the default property names.

✅ Using Modern Value Tuples (C# 7.0 and Later)
Value tuples provide `better syntax and named fields`:
```csharp
var person = (Id: 1, Name: "John", IsActive: true);

Console.WriteLine(person.Id);       // 1
Console.WriteLine(person.Name);     // John
Console.WriteLine(person.IsActive); // True
```

## Returning Multiple Values from a Method
Tuples are commonly used to return multiple values from a function:
```csharp
public (int Id, string Name) GetPerson()
{
    return (1, "Alice");
}

var person = GetPerson();
Console.WriteLine(person.Id);   // 1
Console.WriteLine(person.Name); // Alice
```
✅ Explanation:

- The method returns a named tuple.

- You can access the values using the tuple property names.

## Deconstructing a Tuple
You can `deconstruct` a tuple into separate variables:

```csharp
var (id, name) = GetPerson();

Console.WriteLine(id);   // 1
Console.WriteLine(name); // Alice
```
## Nested Tuples
Tuples can contain other tuples:
```csharp
var nested = (Id: 1, Details: ("Alice", 25));

Console.WriteLine(nested.Details.Item1); // Alice
Console.WriteLine(nested.Details.Item2); // 25
```
## Comparison: `Tuple` vs `ValueTuple`
| Feature             | `Tuple<T1, T2, ...>` | `(T1, T2, ...)` *(ValueTuple)* |
| ------------------- | -------------------- | ------------------------------ |
| Namespace           | `System`             | `System`                       |
| Introduced In       | .NET 4.0             | C# 7.0                         |
| Mutable             | No (Immutable)       | Yes (Mutable)                  |
| Field Names         | Item1, Item2...      | Custom names allowed           |
| Performance         | Reference Type       | Value Type (faster)            |
| Requires Allocation | Yes                  | No (stored on stack)           |

## Example — Practical Use Case
Returning multiple results after processing data:
```csharp
public (int SuccessCount, int FailCount) ProcessData()
{
    int success = 10;
    int fail = 2;
    return (success, fail);
}

var result = ProcessData();
Console.WriteLine($"Success: {result.SuccessCount}, Fail: {result.FailCount}");
/**
Success: 10, Fail: 2
**/
```

## 🧾 Summary
| Concept            | Description                                           |
| ------------------ | ----------------------------------------------------- |
| **Tuple**          | Stores multiple items of different types.             |
| **ValueTuple**     | Struct version of tuple (introduced in C# 7).         |
| **Deconstruction** | Extracts tuple values into variables.                 |
| **Named Elements** | Makes code more readable (e.g., `person.Name`).       |
| **Use Case**       | Commonly used to return multiple values from methods. |

# 🎯 22. Pattern Matching in C#
## What is Pattern Matching?
`Pattern Matching in C#` allows you to `check an object’s type, shape, or value, and extract data from it in a concise and readable way`.

It’s a powerful feature introduced in C# 7.0 and enhanced in later versions (C# 8, 9, 10, and 11).

## Why Use Pattern Matching?
✅ Simplifies conditional logic<br>
✅ Reduces boilerplate `if` / `switch` code<br>
✅ Improves readability and safety (especially with type checks)

### 1. Type Pattern
Used to `check an object’s type and cast it safely` in a single statement.
```csharp
object obj = "Hello World";

if (obj is string message)
{
    Console.WriteLine(message.ToUpper()); // Output: HELLO WORLD
}
```
✅ Explanation:

1. `is string message` checks if obj is a `string`.

2. If true, obj is automatically cast to a string variable named message.
### 2. Constant Pattern
Checks if a variable matches a specific constant value.
```csharp
int number = 10;

if (number is 10)
{
    Console.WriteLine("Number is ten!");
}
```
✅ Explanation:
1. No need for `==`; pattern matching handles the comparison.

### 3. Relational Pattern (C# 9.0+)
Allows comparison operators inside patterns.
```csharp
int age = 25;

if (age is > 18 and < 60)
{
    Console.WriteLine("Adult age group");
}
```
✅ Explanation:
1. Checks if the value of age is greater than 18 and less than 60.

### 4. Logical Patterns (and/or/not)
Combine multiple patterns using logical operators.
```csharp
string role = "Admin";

if (role is "Admin" or "Manager")
{
    Console.WriteLine("Has elevated access");
}

if (role is not "Guest")
{
    Console.WriteLine("User is not a guest");
}
```
✅ Explanation:

1. `or` → Matches any of the patterns

2. `and` → Requires all conditions to match

3. `not` → Negates a pattern

### 5. Switch Pattern Matching
Pattern matching also works with the `switch` expression (modern syntax).
```csharp
object value = 100;

string result = value switch
{
    int i when i > 50 => "Large number",
    int i => "Small number",
    string s => $"String of length {s.Length}",
    null => "Null value",
    _ => "Unknown type"
};

Console.WriteLine(result); // Output: Large number
```
✅ Explanation:

1. Each case can check type and condition using `when`.

2. `_` is the default pattern.

### 6. Property Pattern (C# 8.0+)
Checks the values of object properties directly.
```csharp
var person = new { Name = "Alice", Age = 28 };

if (person is { Name: "Alice", Age: > 18 })
{
    Console.WriteLine("Adult named Alice");
}
```
✅ Explanation:
1. You can check multiple properties in a single expression.

### 7. Positional Pattern (C# 8.0+)
Used with `records` or types that support `deconstruction`.
```csharp
public record Point(int X, int Y);

var point = new Point(10, 20);

if (point is (10, 20))
{
    Console.WriteLine("Point at (10, 20)");
}
```
✅ Explanation:
1. Deconstructs the record and matches values positionally.
### 8. Pattern Matching with `switch` Expression (Modern Syntax)
```csharp
string GetCategory(int age) => age switch
{
    < 13 => "Child",
    >= 13 and < 20 => "Teenager",
    >= 20 and < 60 => "Adult",
    _ => "Senior"
};

Console.WriteLine(GetCategory(25)); // Output: Adult
```
✅ Explanation:
1. This concise `switch` expression makes conditional logic easier to read.

## 🧾 Summary
| Pattern Type           | Description                    | Example                             |
| ---------------------- | ------------------------------ | ----------------------------------- |
| **Type Pattern**       | Checks object type             | `if (obj is string s)`              |
| **Constant Pattern**   | Matches constant value         | `if (num is 10)`                    |
| **Relational Pattern** | Compares numeric values        | `if (age is > 18 and < 60)`         |
| **Logical Pattern**    | Combines patterns              | `if (role is "Admin" or "Manager")` |
| **Property Pattern**   | Matches property values        | `if (person is { Age: > 18 })`      |
| **Positional Pattern** | Matches tuple/record positions | `if (point is (10, 20))`            |
| **Switch Pattern**     | Pattern-based branching        | `x switch { ... }`                  |

# ⚡23. Asynchronous Programming in C#
## What is Asynchronous Programming?
`Asynchronous programming` in C# allows your program to perform `non-blocking operations` — meaning it can continue executing other tasks while waiting for long-running operations (like file I/O, network calls, or database queries) to complete.

✅ In short:<br>
It helps your application stay responsive and efficient, especially in UI apps and web servers.

## Synchronous vs Asynchronous
| Type             | Description                                                                          | Example                                           |
| ---------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------- |
| **Synchronous**  | Tasks run **one after another**. The next task waits for the previous one to finish. | Reading a file blocks until it’s completely read. |
| **Asynchronous** | Tasks run **independently**, allowing other work to continue while waiting.          | Reading a file while UI stays responsive.         |

## 🧩 Example:
### Synchronous Code:
```csharp
public void DownloadFile()
{
    var content = File.ReadAllText("data.txt"); // Blocks here
    Console.WriteLine("File downloaded!");
}
```
### Asynchronous Code:
```csharp
public async Task DownloadFileAsync()
{
    var content = await File.ReadAllTextAsync("data.txt"); // Non-blocking
    Console.WriteLine("File downloaded!");
}
```
✅ Result:<br>
The `await` keyword frees the thread while waiting — allowing other operations to run.

## Core Concepts
1️⃣ async Keyword

Marks a method as `asynchronous` and allows use of the `await` keyword inside it.

2️⃣ await Keyword

Pauses execution of the `async` method until the awaited task completes — without blocking the thread.

3️⃣ Task and Task<TResult>

- Represents an `asynchronous operation`.<br>
- `Task` is used for void-returning async methods;<br>
- `Task<TResult>` returns a result asynchronously.

## Example — Async Method Returning Data
```csharp
public async Task<string> GetDataAsync()
{
    await Task.Delay(2000); // Simulate delay
    return "Data received!";
}

public async Task ExecuteAsync()
{
    string result = await GetDataAsync();
    Console.WriteLine(result);
}
```
### 🧠 Explanation:

- `await Task.Delay(2000)` pauses for 2 seconds without blocking.

- The control returns to the caller, and execution resumes once the task completes.

## Async in UI Applications
In `WPF / WinForms`, async keeps the `UI responsive`:
```csharp
private async void Button_Click(object sender, EventArgs e)
{
    StatusLabel.Text = "Loading...";
    var data = await GetDataAsync();
    StatusLabel.Text = data;
}
```
Without async, the UI would `freeze until the operation completes`.

## Exception Handling in Async Code
Use `try–catch` around awaited calls:
```csharp
try
{
    string data = await GetDataAsync();
    Console.WriteLine(data);
}
catch (Exception ex)
{
    Console.WriteLine($"Error: {ex.Message}");
}
```
✅ Explanation:<br>
Exceptions thrown inside an async method are captured and rethrown when awaited.

## async void vs async Task
| Return Type     | Use Case                            | Notes                                        |
| --------------- | ----------------------------------- | -------------------------------------------- |
| `async void`    | Only for **event handlers**         | Exceptions can’t be awaited or caught easily |
| `async Task`    | Use in most async methods           | Supports `await` and exception handling      |
| `async Task<T>` | For async methods returning a value | Returns result asynchronously                |

## async void in C#
The `async void` return type is used to define asynchronous methods that don’t return a value and can’t be awaited.

### Syntax Example
```csharp
public async void DoWorkAsync()
{
    await Task.Delay(1000);
    Console.WriteLine("Work completed!");
}
```
✅ Explanation:

- The method runs asynchronously.

- The await keyword allows non-blocking execution.

- Since the return type is void, you can’t await this method from the caller.

### When to Use async void
🚫 **Rule of Thumb**: Avoid `async void` except for event handlers.

### ✅ Valid Example — Event Handler
```csharp
private async void Button_Click(object sender, EventArgs e)
{
    await Task.Delay(2000);
    MessageBox.Show("Button clicked!");
}
```
✅ Explanation:

- UI frameworks (like WPF or WinForms) require event handlers to return `void`.

- `async void` works here because event handlers can’t return a `Task`.

### Problems with async void
#### 1. Cannot Be Awaited
You can’t wait for an `async void` method to finish execution.
```csharp
DoWorkAsync(); // Fire-and-forget — no way to know when it completes
```
❌ Not suitable when you need to ensure the operation is done before continuing.

#### 2. Exception Handling is Difficult
Exceptions thrown inside an async void method can’t be caught by the caller.
```csharp
try
{
    DoWorkAsync(); // ❌ Exception here won’t be caught
}
catch (Exception ex)
{
    Console.WriteLine(ex.Message);
}
```
✅ Instead, use async Task so the caller can handle exceptions:

#### 3. Fire-and-Forget Behavior
Because you can’t await it, async void methods execute independently —
you have no control over when or if they complete.<br>
This can cause:
- Race conditions
- Unexpected application state
- Resource leaks in server or background code

### Comparison: async void vs async Task
| Feature                 | `async void`               | `async Task`                  |
| ----------------------- | -------------------------- | ----------------------------- |
| Return Type             | `void`                     | `Task`                        |
| Can be awaited?         | ❌ No                       | ✅ Yes                         |
| Exception Handling      | ❌ Cannot catch in caller   | ✅ Can catch using `try-catch` |
| Usage                   | Event handlers only        | General async methods         |
| Control over completion | ❌ No                       | ✅ Yes                         |
| Best Practice           | Avoid unless event handler | Preferred for async methods   |

### 🧾 Summary - async void
| Concept                | Description                                                    |
| ---------------------- | -------------------------------------------------------------- |
| **`async void`**       | Defines an async method that doesn’t return a value or `Task`. |
| **Main Use Case**      | Event handlers (e.g., UI events).                              |
| **Cannot be awaited**  | The caller cannot wait for it to complete.                     |
| **Exception Handling** | Exceptions can crash the app if unhandled.                     |
| **Best Practice**      | Use only in event handlers; otherwise, prefer `async Task`.    |

## 🧩 Task.WhenAll() in C#
### 📘 Overview
`Task.WhenAll()` is a method in C# used to `run multiple tasks concurrently and wait for all of them to complete`.<br>
It allows asynchronous operations to execute in parallel, improving efficiency when tasks are independent of each other.
### ⚙️ Syntax

```csharp
await Task.WhenAll(task1, task2, task3);
```
or for an array/list of tasks:
```csharp
var tasks = new List<Task> { task1, task2, task3 };
await Task.WhenAll(tasks);
```
### 🧠 How It Works
- `Task.WhenAll()` takes multiple Task objects (or Task<T> objects).

- It starts all tasks simultaneously (if not already started).

- It waits asynchronously until all tasks complete.

- Returns a Task that completes when every provided task has finished execution.

### ✅ Example 1 — Waiting for Multiple Tasks
```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        Task task1 = Task.Delay(2000); // Simulates 2 seconds work
        Task task2 = Task.Delay(1000); // Simulates 1 second work

        Console.WriteLine("Starting tasks...");

        await Task.WhenAll(task1, task2);

        Console.WriteLine("All tasks completed!");
    }
}
/**
Starting tasks...
All tasks completed!
**/
```
### ✅ Example 2 — Returning Values from Tasks
```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        Task<int> task1 = GetNumberAfterDelay(1, 2000);
        Task<int> task2 = GetNumberAfterDelay(2, 1000);

        int[] results = await Task.WhenAll(task1, task2);

        Console.WriteLine($"Results: {string.Join(", ", results)}");
    }

    static async Task<int> GetNumberAfterDelay(int number, int delay)
    {
        await Task.Delay(delay);
        return number * 10;
    }
}
/**
Results: 10, 20
**/
```
### 🧩 Key Points
| Feature                | Description                                          |
| ---------------------- | ---------------------------------------------------- |
| **Parallel Execution** | Runs all tasks concurrently.                         |
| **Wait Behavior**      | Completes when all tasks are done.                   |
| **Return Type**        | `Task` or `Task<T[]>` depending on inputs.           |
| **Error Handling**     | Aggregates exceptions if multiple tasks fail.        |
| **Use Case**           | When you have multiple independent async operations. |

### 🚀 Best Practices
1. ✅ Use Task.WhenAll() instead of multiple awaits when tasks are independent.

2. ❌ Avoid blocking using .Wait() or .Result — always use await.

3. ⚠️ Be mindful of shared resources — ensure thread safety.

4. 💡 Use Task.WhenAny() if you need to proceed when any task finishes first.

### 🧾 Summary
1. Task.WhenAll() lets you efficiently run and await multiple async operations in parallel.

2. It returns only when all tasks are completed or an exception occurs.

3. It helps improve performance in scenarios like fetching data from multiple APIs or performing multiple I/O operations.

## ⚠️ Exception Handling in Task.WhenAll()
### 📘 Overview
If one or more tasks throw exceptions, Task.WhenAll() does not fail immediately — instead, it waits for all tasks to complete first.<br>

After all tasks finish, it throws a single `AggregateException` that contains all the individual exceptions thrown by those tasks.

### 🧠 Key Concept
1. `Task.WhenAll()` collects all exceptions from the tasks that faulted.

2. It then wraps them in an `AggregateException` and rethrows it.

3. You can access each individual exception through the `InnerExceptions` property.

### ✅ Example — Multiple Tasks Throw Exceptions
```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        try
        {
            await Task.WhenAll(Task1(), Task2(), Task3());
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Main catch: {ex.GetType().Name}");
            
            if (ex is AggregateException aggregateEx)
            {
                foreach (var inner in aggregateEx.InnerExceptions)
                {
                    Console.WriteLine($" - Inner exception: {inner.Message}");
                }
            }
        }
    }

    static async Task Task1()
    {
        await Task.Delay(500);
        throw new InvalidOperationException("Task1 failed.");
    }

    static async Task Task2()
    {
        await Task.Delay(1000);
        throw new ArgumentException("Task2 failed.");
    }

    static async Task Task3()
    {
        await Task.Delay(1500);
        throw new NullReferenceException("Task3 failed.");
    }
}
/**
Main catch: AggregateException
 - Inner exception: Task1 failed.
 - Inner exception: Task2 failed.
 - Inner exception: Task3 failed.
**/
```
### 🧩 Why AggregateException?
When working with asynchronous operations:

1. Multiple tasks can fail independently and concurrently.

2. It’s not possible to represent multiple errors with a single exception type.

3. Therefore, .NET wraps them into an AggregateException, which acts as a container for all the thrown exceptions.

### 🔍 Handling Individual Exceptions
You can handle individual exceptions like this:
```csharp
try
{
    await Task.WhenAll(Task1(), Task2());
}
catch (Exception ex)
{
    if (ex is AggregateException ae)
    {
        ae.Handle(inner =>
        {
            if (inner is InvalidOperationException)
            {
                Console.WriteLine("Handled InvalidOperationException.");
                return true;
            }
            return false; // rethrow others
        });
    }
}
```
💡 AggregateException.Handle() allows you to filter and selectively handle specific exception types.

### ⚠️ Important Notes
| Concept                 | Explanation                                                           |
| ----------------------- | --------------------------------------------------------------------- |
| **Waits for all tasks** | Even if one task fails early, others continue running.                |
| **AggregateException**  | Contains all individual exceptions.                                   |
| **InnerExceptions**     | A collection of the actual exceptions thrown.                         |
| **Handle()**            | Used to process or filter specific exceptions.                        |
| **Cancellation**        | If a task is canceled, it’s represented as a `TaskCanceledException`. |

### 🧾 Summary
1. Task.WhenAll() throws after all tasks complete, even if some fail.

2. It aggregates multiple exceptions into one AggregateException.

3. You can use:
        
    1. `InnerExceptions` → to inspect each exception.

    2. `Handle()` → to manage exceptions conditionally.

### ✅ Best Practice Tip:
Always wrap `Task.WhenAll()` inside a try/catch block when tasks might fail independently,and inspect the `AggregateException.InnerExceptions` for detailed error information.

## 🧾 Summary
| Concept                    | Description                                         |
| -------------------------- | --------------------------------------------------- |
| **async / await**          | Enables non-blocking asynchronous programming       |
| **Task / Task<T>**         | Represents ongoing or future asynchronous work      |
| **await**                  | Suspends execution until the awaited task completes |
| **Task.WhenAll / WhenAny** | Run multiple async operations concurrently          |
| **Exception Handling**     | Use `try-catch` with `await`                        |
| **Best Practice**          | Avoid `async void` (except for event handlers)      |

# ⚙️ 24. Understanding the State Machine in C#
## 📘 Overview
In C#, a `state machine` is a compiler-generated construct that allows methods using `async` and `await` to pause and resume execution without blocking a thread.

When you mark a method with `async`, the C# compiler rewrites it into a state machine structure that manages:
- The current execution state
- The continuation points (after each `await`)
- The result or exception handling

## 🧠 Why Do We Need a State Machine?
Asynchronous methods `can pause and resume` execution when awaiting tasks.
However, normal methods in C# cannot "pause in the middle" of execution.

👉 Therefore, the compiler transforms your async method into a `state machine` that:
- Saves the local variables and current state.
- Exits when waiting for a task.
- Resumes from where it left off once the awaited task completes.

### ✅ Example — Async Method
```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        await SayHelloAsync();
    }

    static async Task SayHelloAsync()
    {
        Console.WriteLine("Step 1: Before await");
        await Task.Delay(1000);
        Console.WriteLine("Step 2: After await");
    }
}
```
### 🔍 What Happens Internally
The C# compiler transforms the `SayHelloAsync()` method into a `state machine` class similar to this (simplified for clarity):

```csharp
private sealed class SayHelloAsyncStateMachine : IAsyncStateMachine
{
    public int _state; // Tracks where we are in the method
    public AsyncTaskMethodBuilder _builder; // Builds and manages the async Task
    private TaskAwaiter _awaiter; // Keeps track of the awaited task

    void IAsyncStateMachine.MoveNext()
    {
        try
        {
            if (_state == -1)
            {
                Console.WriteLine("Step 1: Before await");
                _awaiter = Task.Delay(1000).GetAwaiter();
                if (!_awaiter.IsCompleted)
                {
                    _state = 0; // Save state to resume from here
                    _builder.AwaitUnsafeOnCompleted(ref _awaiter, ref this);
                    return;
                }
            }

            if (_state == 0)
            {
                _awaiter.GetResult(); // Resume from await
                Console.WriteLine("Step 2: After await");
            }

            _builder.SetResult(); // Mark task as complete
        }
        catch (Exception ex)
        {
            _builder.SetException(ex);
        }
    }

    void IAsyncStateMachine.SetStateMachine(IAsyncStateMachine stateMachine) { }
}
```

## 🧩 Components of the State Machine
| Component                | Description                                                                    |
| ------------------------ | ------------------------------------------------------------------------------ |
| **`IAsyncStateMachine`** | Interface implemented by compiler-generated state machines.                    |
| **`_state`**             | An integer that tracks where the method last paused (`-1`, `0`, `1`, etc.).    |
| **`_builder`**           | A helper object (`AsyncTaskMethodBuilder`) that builds and completes the Task. |
| **`_awaiter`**           | Holds the awaited task’s state and continuation.                               |
| **`MoveNext()`**         | The core method that runs and resumes the async method execution.              |

## 🔄 Execution Flow
1. The compiler creates an instance of the state machine when the async method is called.
2. It calls the `MoveNext()` method, starting at _state = -1.
3. When an await is encountered:
    1. The current state is saved.
    2. Execution returns to the caller.
4. When the awaited task completes:
    1. The continuation resumes execution from the saved state.
    2. The `MoveNext()` method continues from where it left off.
5. Finally, `_builder.SetResult()` marks the Task as completed.

## ⚙️ State Machine Analogy
Imagine you’re reading a book 📖:
- You pause at page 50 (await Task.Delay()).
- You bookmark that page (_state = 0).
- You resume reading from page 50 once you’re ready (MoveNext() resumes).

That’s exactly how the async state machine works — it “bookmarks” the point of execution.
## 🧾 Summary
| Concept            | Description                                                      |
| ------------------ | ---------------------------------------------------------------- |
| **Purpose**        | Allows async methods to pause/resume execution without blocking. |
| **Generated By**   | C# compiler when using `async` / `await`.                        |
| **Implements**     | `IAsyncStateMachine` interface.                                  |
| **Key Method**     | `MoveNext()` — controls async execution flow.                    |
| **State Tracking** | `_state` variable keeps track of where to resume.                |
| **Continuation**   | Awaits trigger continuation by calling `MoveNext()` again.       |

# ⚙️ 25. Parallel Class in C#
## 📘 Overview
The `Parallel` class in C# is part of the `System.Threading.Tasks` namespace and provides methods to perform `parallel execution of operations on multiple threads`.

It is mainly used for `data parallelism`, where the same operation is performed on different elements of a collection simultaneously.

## 🧠 Key Purpose
The `Parallel` class helps you `run independent tasks concurrently` to:
- Improve performance.
- Utilize multiple CPU cores.
- Reduce execution time for CPU-bound operations.

## ✅ Common Methods in Parallel Class
| Method                   | Description                                                 |
| ------------------------ | ----------------------------------------------------------- |
| **`Parallel.For()`**     | Executes a loop in which iterations run in parallel.        |
| **`Parallel.ForEach()`** | Executes a `foreach` loop where iterations run in parallel. |
| **`Parallel.Invoke()`**  | Executes multiple independent actions concurrently.         |

### 🔹 Example 1 — Using Parallel.For()
```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static void Main()
    {
        Parallel.For(1, 6, i =>
        {
            Console.WriteLine($"Task {i} running on thread {Task.CurrentId}");
        });

        Console.WriteLine("All tasks completed.");
    }
}
/*
Task 2 running on thread 3
Task 4 running on thread 5
Task 1 running on thread 2
Task 3 running on thread 4
Task 5 running on thread 6
All tasks completed.
*/
```
💡 The order is `non-deterministic` — tasks run `concurrently`.

### 🔹 Example 2 — Using Parallel.ForEach()
```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

class Program
{
    static void Main()
    {
        var items = new List<string> { "A", "B", "C", "D" };

        Parallel.ForEach(items, item =>
        {
            Console.WriteLine($"Processing {item} on thread {Task.CurrentId}");
        });
    }
}
```
✅ Ideal for performing the same operation on each element of a collection concurrently.

### 🔹 Example 3 — Using Parallel.Invoke()
```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static void Main()
    {
        Parallel.Invoke(
            () => Console.WriteLine("Task 1 running"),
            () => Console.WriteLine("Task 2 running"),
            () => Console.WriteLine("Task 3 running")
        );

        Console.WriteLine("All actions completed.");
    }
}
```
🔸 Executes multiple independent methods concurrently.

## ⚙️ How It Works
- The Parallel class uses the ThreadPool internally.
- It automatically divides the workload across available processor cores.
- It balances execution dynamically using work-stealing for optimal CPU usage.

## ⚠️ Important Notes
| Concept                | Description                                                                      |
| ---------------------- | -------------------------------------------------------------------------------- |
| **Thread Usage**       | Uses multiple threads from the ThreadPool.                                       |
| **Blocking**           | `Parallel` methods are **blocking** — they don’t return until all tasks finish.  |
| **Order of Execution** | Not guaranteed; tasks may run in any order.                                      |
| **Exceptions**         | If multiple tasks throw exceptions, they are wrapped in an `AggregateException`. |
| **Cancellation**       | Can be supported via `ParallelOptions` and `CancellationToken`.                  |

### 🔹 Example — Using ParallelOptions
```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class Program
{
    static void Main()
    {
        var cts = new CancellationTokenSource();
        var options = new ParallelOptions()
        {
            MaxDegreeOfParallelism = 2, // Limit concurrency
            CancellationToken = cts.Token
        };

        try
        {
            Parallel.For(0, 10, options, i =>
            {
                Console.WriteLine($"Processing {i}");
                if (i == 5)
                    cts.Cancel(); // cancel after 5
            });
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("Operation was cancelled.");
        }
    }
}
```
## 🧾 Summary - Paralle.ForEach
| Feature                | Description                                          |
| ---------------------- | ---------------------------------------------------- |
| **Class**              | `System.Threading.Tasks.Parallel`                    |
| **Used For**           | Parallel loops and concurrent task execution.        |
| **Methods**            | `For`, `ForEach`, `Invoke`.                          |
| **Thread Handling**    | Uses ThreadPool threads automatically.               |
| **Exception Handling** | Throws `AggregateException` for multiple exceptions. |
| **Cancellation**       | Supported via `ParallelOptions`.                     |
| **Blocking Nature**    | Methods block until all tasks are complete.          |

when working with asynchronous operations, you can use `Parallel.ForEachAsync` (introduced in .NET 6) which supports `async delegates`.

## ⚙️ Syntax
```csharp
await Parallel.ForEachAsync(source, async (item, cancellationToken) =>
{
    await SomeAsyncOperation(item);
});
```
## 🧠 Key Points
| Feature             | Description                                         |
| ------------------- | --------------------------------------------------- |
| **Namespace**       | `System.Threading.Tasks`                            |
| **Available since** | .NET 6                                              |
| **Purpose**         | Run asynchronous operations in parallel efficiently |
| **Delegate type**   | `Func<TSource, CancellationToken, ValueTask>`       |
| **Return type**     | `ValueTask` (awaitable)                             |

## 🧩 Example
```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        string[] urls = {
            "https://example.com",
            "https://microsoft.com",
            "https://github.com"
        };

        await Parallel.ForEachAsync(urls, async (url, cancellationToken) =>
        {
            // Simulate an async operation (e.g., HTTP call)
            await Task.Delay(1000, cancellationToken);
            Console.WriteLine($"Processed: {url}");
        });

        Console.WriteLine("All URLs processed!");
    }
}
/*
Processed: https://example.com
Processed: https://microsoft.com
Processed: https://github.com
All URLs processed!
*/
```
(Order may vary since operations run in parallel.)

## ⚠️ Limitations

- You can’t control max degree of parallelism directly (though you can specify via options).
- Not suitable for CPU-bound operations — use regular Parallel.For for those.
- Works only with .NET 6 and later.
### ⚙️ Example with Options
```csharp
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = 3
};

await Parallel.ForEachAsync(urls, options, async (url, token) =>
{
    await Task.Delay(1000, token);
    Console.WriteLine($"Processed with limit: {url}");
});
```
## 🧾 Summary - Parallel.ForEachAsync
| Concept             | Description                                |
| ------------------- | ------------------------------------------ |
| **Method**          | `Parallel.ForEachAsync()`                  |
| **Supports async?** | ✅ Yes                                      |
| **Suitable for**    | I/O-bound async tasks                      |
| **Return Type**     | `ValueTask`                                |
| **Introduced in**   | .NET 6                                     |
| **Advantage**       | Combines parallelism with async efficiency |

# 🧵 26. Multithreading in C#
## 📘 Overview
`Multithreading` in C# is the ability to `execute multiple threads concurrently within a single process`.<br>
It allows programs to perform multiple operations at the same time, improving performance and responsiveness—especially in applications with CPU-bound or I/O-bound tasks.

## ⚙️ What Is a Thread?
A `thread` is the smallest unit of execution within a process.<br>
Every C# application starts with one main thread, and you can create additional threads to perform tasks concurrently.

## 🧩 Why Use Multithreading?
| Benefit                | Description                                                                 |
| ---------------------- | --------------------------------------------------------------------------- |
| **Concurrency**        | Execute multiple operations at once.                                        |
| **Responsiveness**     | Keep UI responsive (important for GUI apps).                                |
| **Performance**        | Utilize multiple CPU cores for better throughput.                           |
| **Separation of Work** | Run independent tasks (like logging, file I/O, or computation) in parallel. |

## 🧠 Example: Creating and Starting a Thread
```csharp
using System;
using System.Threading;

class Program
{
    static void Main()
    {
        Thread thread = new Thread(DoWork);
        thread.Start(); // Start the thread

        Console.WriteLine("Main thread running...");
    }

    static void DoWork()
    {
        Console.WriteLine("Worker thread started...");
        Thread.Sleep(1000); // Simulate some work
        Console.WriteLine("Worker thread finished.");
    }
}
/*
Main thread running...
Worker thread started...
Worker thread finished.
*/
```
## 🧰 Thread Lifecycle
| State             | Description                                                      |
| ----------------- | ---------------------------------------------------------------- |
| **Unstarted**     | Thread created but not yet started.                              |
| **Running**       | Thread is executing code.                                        |
| **WaitSleepJoin** | Thread is blocked (e.g., `Sleep`, waiting on a lock, or `Join`). |
| **Stopped**       | Thread has finished execution.                                   |

## ⚙️ Example: Running Multiple Threads
```csharp
for (int i = 1; i <= 3; i++)
{
    int threadId = i; // capture variable
    Thread thread = new Thread(() =>
    {
        Console.WriteLine($"Thread {threadId} started.");
        Thread.Sleep(500);
        Console.WriteLine($"Thread {threadId} finished.");
    });
    thread.Start();
}
```
## ⚡ Important Concepts in Multithreading
| Concept               | Description                                                                                            |
| --------------------- | ------------------------------------------------------------------------------------------------------ |
| **Thread Pooling**    | Managed threads reused by the .NET runtime (e.g., `ThreadPool.QueueUserWorkItem`).                     |
| **Synchronization**   | Prevents data corruption when multiple threads access shared data (`lock`, `Monitor`, `Mutex`).        |
| **Deadlock**          | When two or more threads are waiting for each other indefinitely.                                      |
| **Race Condition**    | When threads access shared data simultaneously without synchronization, causing unpredictable results. |
| **Context Switching** | Switching CPU execution between threads.                                                               |

## 🔒 Synchronization Example (Using `lock`)
```csharp
class Counter
{
    private int count = 0;
    private readonly object lockObj = new object();

    public void Increment()
    {
        lock (lockObj)
        {
            count++;
            Console.WriteLine($"Count: {count}");
        }
    }
}
```
## 🚀 Modern Alternatives to Threads
| Technique                | Description                                                                          |
| ------------------------ | ------------------------------------------------------------------------------------ |
| **Tasks (`Task` class)** | Higher-level abstraction over threads. Manages scheduling and pooling automatically. |
| **`async` / `await`**    | Simplifies asynchronous programming (especially for I/O-bound operations).           |
| **Parallel Class**       | Executes loops or actions concurrently (`Parallel.For`, `Parallel.ForEach`).         |

## 🧩 Summary
| Feature                | Description                                               |
| ---------------------- | --------------------------------------------------------- |
| **What it is**         | Running multiple threads concurrently within one process. |
| **Main advantage**     | Better performance and responsiveness.                    |
| **Key API**            | `System.Threading` namespace                              |
| **Common issues**      | Race conditions, deadlocks, thread contention.            |
| **Modern replacement** | `Task`, `async/await`, and `Parallel` APIs.               |
