# Entity Framework Core
**Entity Framework Core (EF Core)** is a lightweight, open-source, cross-platform ORM (Object-Relational Mapper) for .NET .<br>

It allows developers to:
- Work with databases using `C# classes`
- Avoid writing most SQL manually
- Perform CRUD operations through `LINQ`
- Map objects to database tables (ORM)

## 🧩 What is ORM in EF Core?
- `ORM` stands for `Object-Relational Mapping`.
- In EF Core, `ORM` is the feature that allows you to **interact with a relational database using C# objects**, instead of writing SQL queries directly.

### 📝 Simple Explanation
ORM converts:
- **C# classes → Database tables**
- **C# objects → Rows in a table**
- **Properties → Columns**
- **LINQ queries → SQL queries executed on the database**

### 🎯 Purpose of ORM in EF Core
EF Core’s ORM helps you:<br>
✔ Avoid writing most SQL manually<br>
✔ Work with data using C# code<br>
✔ Automatically map your classes to database tables<br>
✔ Perform CRUD operations (Create, Read, Update, Delete) easily<br>
✔ Maintain database schema using migrations<br>
✔ Use LINQ instead of SQL<br>

### 🔧 Example – How ORM Works in EF Core
#### C# Entity (Class):
```csharp
public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; }
}

```
ORM maps this class to a table:<br>
**Database Table (Automatically Mapped)**
| Id | Name     |
| -- | -------- |
| 1  | John Doe |

You didn’t create this table manually — EF Core ORM handles it using migrations.

### 🔍 ORM Converts LINQ → SQL
```csharp
var employees = context.Employees
                       .Where(e => e.Name == "John")
                       .ToList();
```
EF Core internally generates SQL like:
```sql
SELECT * FROM Employees WHERE Name = 'John';
```
You write C#, ORM writes SQL on your behalf.

### 🚀 Benefits of ORM in EF Core
✔ Faster development<br>
✔ DB-agnostic (SQL Server, PostgreSQL, MySQL, SQLite)<br>
✔ Cleaner, readable code<br>
✔ Built-in change tracking<br>
✔ Automatic schema management<br>

## 🔹 Key Concepts in EF Core
### 1️⃣ DbContext
`DbContext` represents a session with the database.
- Manages entity objects
- Tracks changes
- Saves data to database
- Provides access to database tables (DbSet)
#### Example:
```csharp
public class AppDbContext : DbContext
{
    public DbSet<Employee> Employees { get; set; }

    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }
}
```
### 2️⃣ Entity
An entity is a C# class mapped to a table in the database.
```csharp
public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Department { get; set; }
}
```
### 3️⃣ DbSet
Represents a **table** in the database.
```csharp
public DbSet<Employee> Employees { get; set; }
```
### 4️⃣ Configuration & Connection String
EF Core connects to the DB via a connection string.
#### Example (for SQL Server in Program.cs / Startup.cs):
```csharp
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));
```
### 5️⃣ Migrations
Migrations are used to **create and update database schema**.
```csharp
# Add migration
dotnet ef migrations add InitialCreate

# Update database
dotnet ef database update
```
Migrations track changes in your model and apply them to the database.
### 6️⃣ CRUD Operations (Basic)
#### Create
```csharp
var employee = new Employee { Name = "John", Department = "HR" };
context.Employees.Add(employee);
context.SaveChanges();
```
#### Read
```csharp
var employees = context.Employees.ToList();
```
With filter:
```csharp
var hrEmployees = context.Employees
                         .Where(e => e.Department == "HR")
                         .ToList();
```
#### Update
```csharp
var emp = context.Employees.Find(1);
emp.Name = "John Doe";
context.SaveChanges();
```
#### Delete
```csharp
var emp = context.Employees.Find(1);
context.Employees.Remove(emp);
context.SaveChanges();
```
### 7️⃣ Change Tracking
EF Core automatically tracks changes of loaded entities.
- Changed fields are detected
- `SaveChanges()` persists the updates

```csharp
emp.Name = "Updated Name"; // EF tracks this change
.context.SaveChanges();
```