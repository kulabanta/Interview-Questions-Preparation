# C# Interview questions Asked
## Q. Can we use `return` statement inside catch block in C#
**Answer:**<br>
- Yes, we absolutely can use a `return` statement inside a `catch`  block in C#.
- When the code hits a `return` statement inside a `catch` block, it evaluates the value to be returned and prepares to exit the method.
- If there is no `finally` block, the method terminates immediately and hands the value back to the caller.
```csharp
public int Divide(int a, int b)
{
    try
    {
        return a / b;
    }
    catch (DivideByZeroException)
    {
        Console.WriteLine("Cannot divide by zero!");
        // Returning a default/fallback value right from the catch
        return 0; 
    }
}
```
- If your `try-catch `structure includes a `finally` block, the `finally` block will always execute before the method actually returns to the caller, even if the `return` statement is inside the `try or catch` block.

```csharp
public string ProcessData()
{
    try
    {
        int x = 10;
        int y = 0;
        int result = x / y; // Throws exception
        return "Success";
    }
    catch (Exception ex)
    {
        Console.WriteLine("1. Inside Catch");
        return "Failed"; // Prepares to return "Failed", but waits...
    }
    finally
    {
        Console.WriteLine("2. Inside Finally");
    }
}
```
```
Output : 
1. Inside Catch
2. Inside Finally
```

## Q. why is string immutable in C# ?
**Answer:**
- In C#, a `string` is immutable, meaning once it is created in memory, its value can never be changed. When you concatenate, replace, or modify a string in your code, you aren't actually altering the original object; the runtime is creating a brand-new string in memory and updating your variable to point to it.

### here are four major reasons why:
#### 1. Memory Efficiency (The String Intern Pool)
- Because strings cannot be changed, the Common Language Runtime (CLR) can safely optimize memory by sharing string instances.

- This process is called `String Interning`. If you have three different variables in your code assigned the literal value "Hello", the CLR doesn't create three separate objects in the heap. It creates "Hello" once in a special area of memory called the `Intern Pool`, and makes all three variables point to that exact same memory address.

- If strings were mutable (changeable), changing one variable to "Help!" would accidentally change the other two variables as well! Immutability makes this memory-saving feature possible.

#### 2. Thread Safety
- Immutable objects are inherently thread-safe. Because a string's state cannot be modified after it's created, multiple threads can safely access and read the exact same string concurrently without needing complex `lock` statements or worrying about race conditions.

#### 3. Security
- Strings are heavily used for sensitive operations: database connection strings, network URLs, and file paths.

- Imagine a scenario where strings were mutable. You could pass a file path to a security validation method. After the validation passes, but right before the file is actually opened, a malicious background thread could mutate the string to point to a sensitive system file instead. By making strings immutable, C# guarantees that the string you validate is the exact same string you execute.

#### 4. Stability in Data Structures (Hash Codes)
- Strings are incredibly common as keys in data structures like `Dictionary<TKey, TValue>` and `HashSet<T>`.

- These collections rely on the object's Hash Code to sort and find data quickly. If you used a string as a dictionary key, and then later modified that string, its Hash Code would change. The dictionary would permanently lose track of where that data was stored, causing memory leaks and bugs. Immutability guarantees that a string's Hash Code will remain identical for its entire lifetime.
## Q. How can I implement a method with same signature which is already implemented by a parent class using an interface ?
- When a child class inherits from a parent class and implements an interface that shares the exact same method signature as the parent, you have a few different ways to handle it depending on the behavior you want.
### Scenario:
```csharp
public interface IWorker 
{ 
    void DoWork(); 
}

public class Parent 
{ 
    public void DoWork() 
    { 
        Console.WriteLine("Parent is working."); 
    } 
}
```
Now, you create a Child class: `public class Child : Parent, IWorker { }`.

How do you implement `DoWork()`? You have three main options.

### 1. Implicit Implementation (Do Nothing)
- If you simply declare the child class and do absolutely nothing, the C# compiler is smart enough to map the parent's method to the interface's requirement.

#### How it works: 
-  The Child class inherits Parent.DoWork(). Because that method's signature exactly matches IWorker.DoWork(), the interface is satisfied automatically.

#### When to use it: 
-  When the parent's logic is exactly what the interface requires, and the child doesn't need to change anything.

### 2. Explicit Interface Implementation
- If you want the Child class to have its own specific implementation for the interface, but leave the Parent method intact, you use `Explicit Interface Implementation`.

#### How it works: 
-  You implement the method by explicitly prefixing it with the interface name. You do not use access modifiers like `public`.

```csharp
public class Child : Parent, IWorker
{
    // Explicitly implementing the interface
    void IWorker.DoWork() 
    {
        Console.WriteLine("Child is working specifically for the interface.");
    }
}

Child myChild = new Child();
myChild.DoWork(); // Output: "Parent is working." (Calls the inherited method)

IWorker workerObj = myChild;
workerObj.DoWork(); // Output: "Child is working specifically for the interface."
```
- An explicitly implemented method is hidden from the normal class instance. 
- To call it, the caller must cast the object to the interface

### 3. Method Hiding (Using the new Keyword)
- If you want the `Child` class to completely replace the parent's method for all standard calls, but the parent's method wasn't marked as `virtual`, you can use the `new` keyword to "hide" or "shadow" the parent's method.

#### How it works: 
-  You declare the method in the child class with the `new` keyword. This method will automatically satisfy the IWorker interface.

#### When to use it: 
- When you don't control the Parent class (e.g., it's from a third-party library), it isn't virtual, and you need the child to have completely different default behavior.

```csharp
public class Child : Parent, IWorker
{
    // Hiding the parent's method
    public new void DoWork() 
    {
        Console.WriteLine("Child is working (hiding parent).");
    }
}
```
### 4. Overriding (If the Parent is virtual)
- If you actually own the Parent class code, the cleanest Object-Oriented approach is to mark the parent's method as `virtual` and `override` it in the child.
- Like method hiding, this overridden method will automatically satisfy the IWorker interface requirement.
```csharp
// In Parent class:
public virtual void DoWork() { ... }

// In Child class:
public override void DoWork() 
{ 
    Console.WriteLine("Child is overriding parent."); 
}
```
## Q. How to run multiple threads in sequence ?
Multiple approaches :

1. async / await : modern approach
2. Task.ContinueWith()
3. Thread.Join() 

## 