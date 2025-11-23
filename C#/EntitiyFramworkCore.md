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
## #️⃣ CRUD Operations in Entity Framework Core
CRUD stands for:
- Create
- Read
- Update
- Delete

These are the basic operations used to interact with a database.<br>
EF Core allows you to perform CRUD using C# objects and LINQ, without manually writing SQL.
### 🏗️ 1. Setup (DbContext & Entity Example)
```csharp
public class AppDbContext : DbContext
{
    public DbSet<Employee> Employees { get; set; }

    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }
}

public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Department { get; set; }
}
```
### #️⃣ 2. CREATE — Insert Data
To insert a new record, you:
1. Create a C# object
2. Add it to the DbSet
3. Call `SaveChanges()` to persist it to the database
#### ✔ Example
```csharp
var employee = new Employee
{
    Name = "John Doe",
    Department = "HR"
};

context.Employees.Add(employee);
context.SaveChanges();
```
**What EF Core does internally**<br>
EF Core generates SQL like:
```sql
INSERT INTO Employees (Name, Department)
VALUES ('John Doe', 'HR');
```
**Notes:**
- `Add()` marks the entity state as Added
- `SaveChanges()` executes the SQL statement

### #️⃣ 3. READ — Fetch Data
EF Core uses `LINQ` to query the database.
#### ✔ 3.1 Fetch all records
```csharp
var employees = context.Employees.ToList();
```
#### ✔ 3.2 Fetch with filter (WHERE clause)
```csharp
var hrEmployees = context.Employees
                         .Where(e => e.Department == "HR")
                         .ToList();
```
#### ✔ 3.3 Fetch single record by Id (Primary key)
```csharp
var employee = context.Employees.Find(1);
```
`Find()`
- Uses the primary key
- Checks in-memory tracked entities first
- If not found, it queries the database
#### ✔ 3.4 Ordering, Projection, and Joins
```csharp
var names = context.Employees
                   .OrderBy(e => e.Name)
                   .Select(e => e.Name)
                   .ToList();
```
### #️⃣ 4. UPDATE — Modify Existing Data
Steps:
1. Retrieve the entity
2. Modify its properties
3. Call `SaveChanges()`

EF Core automatically detects the changes (Change Tracking).

#### ✔ Example
```csharp
var emp = context.Employees.Find(1);

emp.Name = "John Smith";
emp.Department = "Finance";

context.SaveChanges();
```
**What EF Core does internally**

Generates SQL like:
```sql
UPDATE Employees 
SET Name = 'John Smith', Department = 'Finance'
WHERE Id = 1;
```
**Notes:**
- No need to call any Update() unless entity is detached
- EF tracks the entity and knows what fields changed
#### ✔ If entity is not tracked (Detached Update)
```csharp
var emp = new Employee
{
    Id = 1,
    Name = "New Name",
    Department = "IT"
};

context.Employees.Update(emp);
context.SaveChanges();
```
### #️⃣ 5. DELETE — Remove Data
To delete a record:
#### ✔ Example
```csharp
var emp = context.Employees.Find(1);

context.Employees.Remove(emp);
context.SaveChanges();
```
**Internal SQL**
```sql
DELETE FROM Employees WHERE Id = 1;
```
#### ✔ Delete without fetching (Detached)
```csharp
context.Employees.Remove(new Employee { Id = 1 });
context.SaveChanges();
```
Only `primary key` is needed.
## 🎯 CRUD Summary Table
| Operation  | Method                           | Notes             |
| ---------- | -------------------------------- | ----------------- |
| **Create** | `Add()`                          | Insert new entity |
| **Read**   | LINQ (`Where`, `Find`, `ToList`) | Fetch data        |
| **Update** | Modify and call `SaveChanges()`  | Tracks changes    |
| **Delete** | `Remove()`                       | Deletes entity    |
## 🧠 EF Core Change Tracking (Important!)

EF Core automatically tracks entities retrieved from the database.
- When you modify a property, EF Core detects the change
- When you call `SaveChanges()`, only changed fields are updated

## ⭐ Best Practices for CRUD
✔ Use async versions (`AddAsync`, `ToListAsync`)<br>
✔ Use `Find()` for primary key lookups<br>
✔ Prefer LINQ projections instead of loading entire entity<br>
✔ Use tracking only when needed<br>
✔ For high-performance reads → use `.AsNoTracking()`<br>
Example:
```csharp
var employees = context.Employees
                       .AsNoTracking()
                       .Where(e => e.Department == "HR")
                       .ToList();

```
## #️⃣ LINQ Queries in C# – Detailed Explanation
**LINQ (Language Integrated Query)** is a powerful querying feature in C#.<br>
It allows you to query collections, databases (via EF Core), XML, and more using C# syntax instead of SQL.<br>
LINQ is type-safe, readable, and integrated directly into the language.
### 🔹 What is LINQ?
**LINQ (Language Integrated Query)** is a set of C# features that lets you query data using:
- query syntax (SQL-like), or
- method syntax (uses extension methods like Where, Select, etc.)

It works with:
- In-memory collections (List<T>, arrays)
- Databases (via Entity Framework Core)
- XML documents
- Remote services
- Files

### 🔹 LINQ Syntax Types
LINQ has two styles:
#### 1️⃣ Query Syntax (SQL-like)
```csharp
var result = from e in employees
             where e.Department == "IT"
             select e;
```
#### 2️⃣ Method Syntax (Most Common)
```csharp
var result = employees
              .Where(e => e.Department == "IT")
              .ToList();
```
EF Core internally converts method syntax into SQL queries.

### #️⃣ Core LINQ Methods (with Examples)
Below are the most important LINQ methods you’ll use every day.

#### 1️⃣ Where – Filtering Data
Filters elements based on a condition.
```csharp
var itEmployees = employees
                  .Where(e => e.Department == "IT")
                  .ToList();
```
#### 2️⃣ Select – Projection (Transform Output)
Chooses what fields to return.
```csharp
var names = employees
            .Select(e => e.Name)
            .ToList();
```
Multiple fields:
```csharp
var info = employees
           .Select(e => new { e.Name, e.Department })
           .ToList();
```
#### 3️⃣ OrderBy / OrderByDescending
Sort data.
```csharp
var ordered = employees
              .OrderBy(e => e.Name)
              .ToList();
```
Descending:
```csharp
var ordered = employees
              .OrderByDescending(e => e.Salary)
              .ToList();
```
#### 4️⃣ ThenBy / ThenByDescending
Used after `OrderBy` for secondary sorting.
```csharp
var sorted = employees
             .OrderBy(e => e.Department)
             .ThenBy(e => e.Name)
             .ToList();
```
#### 5️⃣ GroupBy – Group Elements
```csharp
var groups = employees
             .GroupBy(e => e.Department)
             .Select(g => new 
             {
                 Department = g.Key,
                 Count = g.Count()
             })
             .ToList();
```
#### 6️⃣ Join – Combine 2 Collections
```csharp
var query = employees
           .Join(departments,
                e => e.DepartmentId,
                d => d.Id,
                (e, d) => new { e.Name, d.DepartmentName })
           .ToList();
```
Works like SQL INNER JOIN.
#### 7️⃣ Distinct – Remove duplicates
```csharp
var uniqueDepartments = employees
                        .Select(e => e.Department)
                        .Distinct()
                        .ToList();
```
#### 8️⃣ First / FirstOrDefault
Returns the first match.
```csharp
var emp = employees.First(e => e.Id == 5);
```
If no match:
```csharp
var emp = employees.FirstOrDefault(e => e.Id == 5);
```
`FirstOrDefault` is safer — returns null if not found.

#### 9️⃣ Single / SingleOrDefault
Used when you expect `exactly one` match.
```csharp
var admin = users.Single(u => u.Email == "admin@test.com");
```
Throws exception if more than one result exists.
#### 🔟 Any – Check Whether At Least One Element Exists
```csharp
bool hasIT = employees.Any(e => e.Department == "IT");
```
#### 1️⃣1️⃣ Count / Sum / Min / Max / Average (Aggregation)
Count:
```csharp
int total = employees.Count();
```
Sum salaries:
```csharp
decimal totalSalary = employees.Sum(e => e.Salary);
```
Max:
```csharp
var highest = employees.Max(e => e.Salary);
```
## #️⃣ LINQ Operations in Entity Framework Core
LINQ is the primary way to query databases in EF Core.<br>
These operations are translated into SQL queries and executed on the database.

EF Core supports most LINQ operators, but some run only server-side (good), while others run client-side (bad for performance).

### 1️⃣ Filtering Operators
✔ Where() — Filter rows
```csharp
var employees = context.Employees
                      .Where(e => e.Department == "IT")
                      .ToList();
```
Equivalent SQL:
```sql
SELECT * FROM Employees WHERE Department = 'IT';
```
### 2️⃣ Projection Operators
✔ Select() — Choose specific columns
```csharp
var result = context.Employees
                   .Select(e => new { e.Name, e.Salary })
                   .ToList();

```
Used to create DTOs, anonymous objects, or transform data.

### 3️⃣ Sorting Operators
✔ OrderBy()
```csharp
var sorted = context.Employees
                    .OrderBy(e => e.Name)
                    .ToList();  
```
✔ OrderByDescending()
```csharp
var sorted = context.Employees
                    .OrderByDescending(e => e.Salary)
                    .ToList();
```
✔ ThenBy(), ThenByDescending()
- Used for multi-level sorting.

### 4️⃣ Aggregation Operators
Used to compute values like count, sum, average.
```csharp
var totalEmployees = context.Employees.Count();
var totalSalary = context.Employees.Sum(e => e.Salary);
var maxSalary = context.Employees.Max(e => e.Salary);
var minSalary = context.Employees.Min(e => e.Salary);
var avgSalary = context.Employees.Average(e => e.Salary);
```
Translated to SQL aggregate functions.

### 5️⃣ Grouping Operators
✔ GroupBy() — Create groups like SQL GROUP BY
```csharp
var groups = context.Employees
                    .GroupBy(e => e.Department)
                    .Select(g => new { 
                        Department = g.Key, 
                        Count = g.Count() 
                    })
                    .ToList();
```
### 6️⃣ Join Operators
✔ Join() – Inner join
```csharp
var result = context.Employees
                   .Join(context.Departments,
                         e => e.DepartmentId,
                         d => d.Id,
                         (e, d) => new { e.Name, d.DepartmentName })
                   .ToList();
```
✔ GroupJoin() – Left join
```csharp
var result = context.Departments
                   .GroupJoin(context.Employees,
                              d => d.Id,
                              e => e.DepartmentId,
                              (d, empGroup) => new { d.DepartmentName, Employees = empGroup })
                   .ToList();
```
### 7️⃣ Element Operators
Used to retrieve single items.

✔ First(), FirstOrDefault()
```csharp
var emp = context.Employees.FirstOrDefault(e => e.Id == 5);
```
✔ Single(), SingleOrDefault()
```csharp
var admin = context.Users.Single(u => u.Email == "admin@test.com");
```
✔ Find() — fastest for primary keys
```csharp
var emp = context.Employees.Find(1);
```

### 8️⃣ Quantifier Operators

✔ Any() — Checks if at least one record exists
```csharp
bool exists = context.Employees.Any(e => e.Department == "HR");
```
✔ All() — Checks if all records match a condition
```csharp
bool allHighSalary = context.Employees.All(e => e.Salary > 50000);
```

### 9️⃣ Set Operators
✔ Distinct()
```csharp
var departments = context.Employees
                         .Select(e => e.Department)
                         .Distinct()
                         .ToList();
```
✔ Union(), Intersect(), Except()
```csharp
var result = list1.Union(list2).ToList();
```

Note: Used mostly in advanced EF queries.

### 1️⃣1️⃣ Tracking & No-Tracking Queries
✔ Tracking (default)

- EF tracks entities for changes.

✔ No Tracking — Better performance for read-only
```csharp
var data = context.Employees
                  .AsNoTracking()
                  .ToList();
```
### 1️⃣2️⃣ Pagination Operators
✔ Skip() and Take() — Used for paging
```csharp
var pageData = context.Employees
                      .Skip(10)
                      .Take(10)
                      .ToList();
```

SQL equivalent:
```sql
OFFSET 10 ROWS FETCH NEXT 10 ROWS ONLY;
```

| Method          | Purpose                 | Example             |
| --------------- | ----------------------- | ------------------- |
| **Skip(n)**     | Skips first *n* records | `Skip(10)`          |
| **Take(n)**     | Takes next *n* records  | `Take(10)`          |
| **Skip + Take** | Used for pagination     | `Skip(10).Take(10)` |

#### Example: Page Number & Page Size
```csharp
int pageNumber = 2;
int pageSize = 10;

var students = context.Students
                      .OrderBy(s => s.Id)
                      .Skip((pageNumber - 1) * pageSize)
                      .Take(pageSize)
                      .ToList();
```
##### 📌 Explanation:

1. Page 1 → skip 0, take 10

2. Page 2 → skip 10, take 10

3. Page 3 → skip 20, take 10

##### ✔️ SQL Translation in EF Core

- EF Core converts Skip() + Take() into SQL using OFFSET and FETCH.

Example SQL:
```sql
SELECT * FROM Students
ORDER BY Id
OFFSET 10 ROWS
FETCH NEXT 10 ROWS ONLY;
```
## 1️⃣3️⃣ Conversion Operators
✔ ToList()

- Executes the query and loads results into memory.

✔ ToArray(), ToDictionary()
```csharp
var dict = context.Employees
                  .ToDictionary(e => e.Id);
```

### ⭐ Which LINQ operations run on the database in EF Core?
| Category              | Supported Server-Side?                             |
| --------------------- | -------------------------------------------------- |
| Filtering             | ✅ Yes                                              |
| Projection            | ✅ Yes                                              |
| Sorting               | ✅ Yes                                              |
| Aggregation           | ✅ Yes                                              |
| Grouping              | ⚠️ Mostly, but complex grouping may be client-side |
| Joins                 | ✅ Yes                                              |
| Navigation properties | EF Core translates includes                        |
| Custom functions      | ❌ No (client-side)                                 |
