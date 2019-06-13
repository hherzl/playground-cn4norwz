# Scaffolding ASP.NET Core 2 with CatFactory

## Introduction

## What Is CatFactory?

CatFactory is a scaffolding engine for .NET Core built with C#.

## How does it Works?

The concept behind CatFactory is to import an existing database from SQL Server instance and then to scaffold a target technology.

We can also replace the database from SQL Server instance with an in-memory database.

The flow to import an existing database is:

1. Create Database Factory
2. Import Database
3. Create instance of Project (Entity Framework Core, Dapper, etc)
4. Build Features (One feature per schema)
5. Scaffold objects, these methods read all objects from database and create instances for code builders

Currently, the following technologies are supported:

+ [`Entity Framework Core`](https://github.com/hherzl/CatFactory.EntityFrameworkCore)
+ [`ASP.NET Core`](https://github.com/hherzl/CatFactory.AspNetCore)
+ [`Dapper`](https://github.com/hherzl/CatFactory.Dapper)

This package is the core for child packages, additional packages have created with this naming convention: CatFactory.**PackageName**.

* CatFactory.SqlServer
* CatFactory.NetCore
* CatFactory.EntityFrameworkCore
* CatFactory.AspNetCore
* CatFactory.Dapper

## Roadmap

There will be a lot of improvements for CatFactory on road:

* Scaffolding Services Layer
* Dapper Integration for ASP.NET Core
* MD files
* Scaffolding C# Client for ASP.NET Web API
* Scaffolding Unit Tests for ASP.NET Core
* Scaffolding Integration Tests for ASP.NET Core
* Scaffolding Angular

## Concepts behind CatFactory

### Database Type Map

One of things I don't like to get equivalent between SQL data type for CLR is use magic strings, after of review the more "fancy" way to resolve a type equivalence is to have a class that allows to know the equivalence between SQL data type and CLR type.

This concept was created from this matrix: [`SQL Server Data Type Mappings`](https://docs.microsoft.com/en-us/dotnet/framework/data/adonet/sql-server-data-type-mappings).

Using this matrix as reference, now CatFactory has a class named DatabaseTypeMap. Database class contains a property with all mappings named DatebaseTypeMaps, so this property is filled by Import feature for SQL Server package.

```csharp
public class DatabaseTypeMap
{
    public string DatabaseType { get; set; }

    public bool AllowsLengthInDeclaration { get; set; }

    public bool AllowsPrecInDeclaration { get; set; }

    public bool AllowsScaleInDeclaration { get; set; }

    public string ClrFullNameType { get; set; }

    public bool HasClrFullNameType { get; }

    public string ClrAliasType { get; set; }

    public bool HasClrAliasType { get; }

    public bool AllowClrNullable { get; set; }

    public DbType DbTypeEnum { get; set; }

    public bool IsUserDefined { get; set; }

    public string ParentDatabaseType { get; set; }

    public string Collation { get; set; }
}
```

DatabaseTypeMap is the class to represent database type definition, for database instance we need to create a collection of DatabaseTypeMap class to have a matrix to resolve data types.

Suppose there is a class with name DatabaseTypeMapList, this class has a property to get data types. Once we have imported an existing database we can resolve data types:

Resolve without extension methods:

```csharp
// Get mappings
var dataTypes = database.DatabaseTypeMaps;

// Resolve CLR type
var mapsForString = dataTypes.Where(item => item.ClrType == typeof(string)).ToList();

// Resolve SQL Server type
var mapForVarchar = dataTypes.FirstOrDefault(item => item.DatabaseType == "varchar");
```

Resolve with extension methods:

```csharp
// Get database type
var varcharDataType = database.ResolveType("varchar");

// Resolve CLR
var mapForVarchar = varcharDataType.GetClrType();
```

SQL Server allows to define data types, suppose the database instance has a data type defined by user with name Flag, Flag data type is a bit, bool in C#.

Import method retrieve user data types, so in DatabaseTypeMaps collection we can search the parent data type for Flag:

### Project Selection

A project selection is a limit to apply settings for objects that match with pattern.

GlobalSelection is the default selection for project, contains a default instance of settings.

Patterns:

|Pattern|Scope|
|-------|-----|
|Sales.OrderHeader|Applies for specific object with name Sales.OrderHeader|
|Sales.\*|Applies for all objects inside of Sales schema|
|\*.OrderHeader|Applies for all objects with name Order with no matter schema|
|\*.\*|Applies for all objects, this is the global selection|

Sample:

```csharp
// Apply settings for Project
project.GlobalSelection(settings =>
{
    settings.ForceOverwrite = true;
    settings.AuditEntity = new AuditEntity("CreationUser", "CreationDateTime", "LastUpdateUser", "LastUpdateDateTime");
    settings.ConcurrencyToken = "Timestamp";
});

// Apply settings for specific object
project.Select("Sales.OrderHeader", settings =>
{
    settings.EntitiesWithDataContracts = true;
});
```

### Event Handlers to Scaffold

In order to provide a more flexible way to scaffold, there are two delegates in CatFactory, one to perform an action before of scaffolding and another one to handle and action after of scaffolding.

```csharp
// Add event handlers to before and after of scaffold

project.ScaffoldingDefinition += (source, args) =>
{
    // Add code to perform operations with code builder instance before to create code file
};

project.ScaffoldedDefinition += (source, args) =>
{
    // Add code to perform operations after of create code file
};
```

## Packages

	CatFactory
	CatFactory.SqlServer
	CatFactory.NetCore
	CatFactory.EntityFrameworkCore
	CatFactory.AspNetCore
	CatFactory.Dapper
	CatFactory.TypeScript

You can check the download statistics for **CatFactory** packages in [`NuGet Gallery`](https://www.nuget.org/packages?q=CatFactory).

## Background

Generate code is a common task in software developer, most developers write a "code generator" in their lives.

Using Entity Framework 6.x, I worked with EF wizard and it's a great tool even with limitations like:

	No scaffolding for Fluent API
	No scaffolding for Repositories
	No scaffolding for Unit of Work
	Custom scaffolding is so complex or in some cases impossible

With *Entity Framework Core*, I worked with command line to scaffold from existing database, EF Core team provided a great tool with command line but there are still the same limitations above.

So, **CatFactory** pretends to solve those limitations and provide a simple way to scaffold *Entity Framework Core*.

*StringBuilder* was used to scaffold a class or interface in older versions of **CatFactory** but some years ago, there was a change about how to scaffold a definition (class or interface), **CatFactory** allows to define the structure for class or interface in a simple and clear way, then use an instance of *CodeBuilder* to scaffold in C#.

Let's start with scaffold a class in C#:

```csharp
var definition = new CSharpClassDefinition
{
    Namespace = "OnlineStore.DomainDrivenDesign",
    AccessModifier = AccessModifier.Public,
    Name = "StockItem",
    Properties =
    {
        new PropertyDefinition(AccessModifier.Public, "string", "GivenName")
        {
            IsAutomatic = true
        },
        new PropertyDefinition(AccessModifier.Public, "string", "MiddleName")
        {
            IsAutomatic = true
        },
        new PropertyDefinition(AccessModifier.Public, "string", "Surname")
        {
            IsAutomatic = true
        },
        new PropertyDefinition(AccessModifier.Public, "string", "FullName")
        {
            IsReadOnly = true,
            GetBody =
            {
                new CodeLine(" return GivenName + (string.IsNullOrEmpty
                  (MiddleName) ? \"\" : \" \" + MiddleName) + \" \" + Surname)")
            }
        }
    }
};

CSharpCodeBuilder.CreateFiles("C:\\Temp", string.Empty, true, definition);
```

This is the output code:

```csharp
namespace OnlineStore.DomainDrivenDesign
{
	public class StockItem
	{
		public string GivenName { get; set; }

		public string MiddleName { get; set; }

		public string Surname { get; set; }

		public string FullName
			=> GivenName + (string.IsNullOrEmpty(MiddleName) ? "" : " " + 
                            MiddleName) + " " + Surname;

	}
}
```

To create an object definition like class or interface, these types can be used:

	EventDefinition
	FieldDefinition
	ClassConstructorDefinition
	FinalizerDefinition
	IndexerDefinition
	PropertyDefinition
	MethodDefinition

Types like *ClassConstructorDefinition*, *FinalizerDefinition*, *IndexerDefinition*, *PropertyDefinition* and *MethodDefinition* can have code blocks, these blocks are arrays of ILine.

*ILine* interface allows to represent a code line inside of code block, there are different types for lines:

	1. CodeLine
	2. CommentLine
	3. EmptyLine
	4. PreprocessorDirectiveLine
	5. ReturnLine
	6. TodoLine

Let's create a class with methods:

```csharp
var classDefinition = new CSharpClassDefinition
{
 Namespace = "OnlineStore.BusinessLayer",
 AccessModifier = AccessModifier.Public,
 Name = "WarehouseService",
 Fields =
 {
  new FieldDefinition("OnlineStoreDbContext", "DbContext")
  {
   IsReadOnly = true
  }
 },
 Constructors =
 {
  new ClassConstructorDefinition
  {
   AccessModifier = AccessModifier.Public,
   Parameters =
   {
    new ParameterDefinition("OnlineStoreDbContext", "dbContext")
   },
   Lines =
   {
    new CodeLine("DbContext = dbContext;")
   }
  }
 },
 Methods =
 {
  new MethodDefinition
  {
   AccessModifier = AccessModifier.Public,
   Type = "IListResponse<StockItem>",
   Name = "GetStockItems",
   Lines =
   {
    new TodoLine(" Add filters"),
    new CodeLine("return DbContext.StockItems.ToList();")
   }
  }
 }
};

CSharpCodeBuilder.CreateFiles("C:\\Temp", string.Empty, true, definition);
```

This is the output code:

```csharp
namespace OnlineStore.BusinessLayer
{
 public class WarehouseService
 {
  private readonly OnlineStoreDbContext DbContext;

  public WarehouseService(OnlineStoreDbContext dbContext)
  {
   DbContext = dbContext;
  }

  public IListResponse<StockItem> GetStockItems()
  {
   // todo:  Add filters
   return DbContext.StockItems.ToList();
  }
 }
}
```

Now let's refact an interface from class:

```csharp
var interfaceDefinition = classDefinition.RefactInterface();

CSharpCodeBuilder.CreateFiles(@"C:\Temp", string.Empty, true, interfaceDefinition);
```

This is the output code:

```csharp
public interface IWarehouseService
{
	IListResponse<StockItem> GetStockItems();
}
```

I know some developers can reject this design alleging there is a lot of code to scaffold a simple class with 4 properties but keep in mind **CatFactory**'s way looks like a "clear" transcription of definitions.

**CatFactory.NetCore** uses the model from CatFactory to allow scaffold C# code, so the question is: What is **CatFactory.Dapper** package?

Is a package that allows to scaffold Dapper using scaffolding engine provided by **CatFactory**.

## Prerequisites

### Skills

    C#
	Object Relational Mapping
    Design Patterns (Repository and Unit of Work)

### Software

    .NET Core
    Visual Studio or VS Code
    SQL Server

## Using the code

Please follow these steps to scaffold Entity Framework Core with CatFactory:

### Step 01 - Create sample database

Take a look for sample database to understand each component in architecture. In this database there are 4 schemas: Dbo, HumanResources, Warehouse and Sales.

Each schema represents a division on store company, keep this in mind because all code is designed following this aspect; at this moment this code only implements features for Production and Sales schemas.

All tables have a primary key with one column and have columns for creation, last update and concurrency token.

#### Tables

|Schema|Name|
|------|----|
|dbo|ChangeLog|
|dbo|ChangeLogExclusion|
|dbo|Country|
|dbo|CountryCurrency|
|dbo|Currency|
|dbo|EventLog|
|HumanResources|Employee|
|HumanResources|EmployeeAddress|
|HumanResources|EmployeeEmail|
|Sales|Customer|
|Sales|OrderDetail|
|Sales|OrderHeader|
|Sales|OrderStatus|
|Sales|PaymentMethod|
|Sales|Shipper|
|Warehouse|Location|
|Warehouse|Product|
|Warehouse|ProductCategory|
|Warehouse|ProductInventory|

You can found the scripts for database in this link: [`Online Store Database Scripts on GitHub`](https://github.com/hherzl/EntityFrameworkCore2ForEnterprise/tree/master/Resources/Database).

Please remember: This is a sample database, only for demonstration of concepts.

### Step 02 - Create project

Create a console application for .NET Core, in some cases you can add one project to your existing solution but with some name or sufix that indicates it's a project to scaffold, for example: *OnLineStore.CatFactory.EntityFrameworkCore*.

Add the following packages for your project:

|Name|Version|Description|
|----|-------|-----------|
|**CatFactory.SqlServer**|1.0.0-beta-sun-build27|Provides import feature for SQL Server databases|
|**CatFactory.EntityFrameworkCore**|1.0.0-beta-sun-build25|Provides scaffold for Entity Framework Core|

#### Install with Nuget UI

	Right Click on Project
	Select Manage Nuget Packages
	Select Browse tab

#### Install with Package Manager Console

	Go to Tools menu
	Select Nuget Package Manager
	Select Package Manager Console

Run this command: **Install-Package CatFactory.EntityFrameworkCore -Version 1.0.0-beta-sun-build35**.

Save all changes and build the project.

### Step 03 - Add Code to scaffold

Once We have the packages installed on our project, We can add the following code in *Main* method:

```csharp
// Create database factory
var databaseFactory = new SqlServerDatabaseFactory
{
    DatabaseImportSettings = new DatabaseImportSettings
    {
        ConnectionString = "server=(local);database=OnlineStore;integrated security=yes;",
        ImportTableFunctions = true,
        Exclusions =
        {
            "dbo.sysdiagrams",
            "dbo.fn_diagramobjects"
        }
    }
};

// Import database
var database = databaseFactory.Import();

// Create instance of Entity Framework Core project
var project = new EntityFrameworkCoreProject
{
    Name = "OnlineStore.Core",
    Database = database,
    OutputDirectory = @"C:\Projects\OnlineStore.Core"
};

// Apply settings for Entity Framework Core project
project.GlobalSelection(settings =>
{
    settings.ForceOverwrite = true;
    settings.ConcurrencyToken = "Timestamp";
    settings.AuditEntity = new AuditEntity
    {
        CreationUserColumnName = "CreationUser",
        CreationDateTimeColumnName = "CreationDateTime",
        LastUpdateUserColumnName = "LastUpdateUser",
        LastUpdateDateTimeColumnName = "LastUpdateDateTime"
    };
});

project.Selection("Sales.OrderHeader", settings => settings.EntitiesWithDataContracts = true);

// Build features for project, group all entities by schema into a feature
project.BuildFeatures();

// Add event handlers to before and after of scaffold

project.ScaffoldingDefinition += (source, args) =>
{
    // Add code to perform operations with code builder instance before to create code file
};

project.ScaffoldedDefinition += (source, args) =>
{
    // Add code to perform operations after of create code file
};

// Scaffolding =^^=
project
    .ScaffoldEntityLayer()
    .ScaffoldDataLayer();
```

### Step 04 - Create Console Project

The following code shows how to use the scaffolded code, if We want to use the code with ASP.NET Web API project, We'll need to set up dependency injection and anothers things, please check in related links for more information.

Now We can go to output directory and create a console project for .NET Core, select code snippet according to your case:

#### Get All

```csharp
// Create options for DbContext instance
var options = new DbContextOptionsBuilder<OnlineStoreDbContext>()
    .UseSqlServer("server=(local);database=OnlineStore;integrated security=yes;")
    .Options;

// Create DbContext instance
var dbContext = new OnlineStoreDbContext(options);

using (var repository = new SalesRepository(dbContext))
{
    // Get "in memory" query
    var query = repository.GetOrderHeaders();
    
    // Get paging info
    var totalItems = await query.CountAsync();
    var pageSize = 25;
    var pageNumber = 1;
    var pageCount = totalItems / pageSize;
    
    // Retrieve list from database
    var list = await query.Paging(pageSize, pageNumber).ToListAsync();
}
```

#### Get by Key

```csharp
// Create options for DbContext instance
var options = new DbContextOptionsBuilder<OnlineStoreDbContext>()
    .UseSqlServer("server=(local);database=OnlineStore;integrated security=yes;")
    .Options;

// Create DbContext instance
var dbContext = new OnlineStoreDbContext(options);

using (var repository = new SalesRepository(dbContext))
{
    // Get entity by id
    var entity = await repository.GetOrderHeaderAsync(new OrderHeader(1));
}
```

#### Get by Unique (If table contains)

```csharp
// Create options for DbContext instance
var options = new DbContextOptionsBuilder<OnlineStoreDbContext>()
    .UseSqlServer("server=(local);database=OnlineStore;integrated security=yes;")
    .Options;

// Create DbContext instance
var dbContext = new OnlineStoreDbContext(options);

using (var repository = new ProductionRepository(dbContext))
{
    // Get entity by name
    var entity = await repository.GetProductByProductNameAsync(new Product { ProductName = "The King of Fighters XIV" });
}
```

#### Add

```csharp
// Create options for DbContext instance
var options = new DbContextOptionsBuilder<OnlineStoreDbContext>()
    .UseSqlServer("server=(local);database=OnlineStore;integrated security=yes;")
    .Options;

// Create DbContext instance
var dbContext = new OnlineStoreDbContext(options);

using (var repository = new SalesRepository(dbContext))
{
    // Create instance for entity
    var entity = new OrderHeader();
    
    // Set values for properties
    // e.g. entity.Total = 29.99m;
    
    // Add entity in database
    repository.Add(entity);

    await respository.CommitChangesAsync();
}
```

#### Update

```csharp
// Create options for DbContext instance
var options = new DbContextOptionsBuilder<OnlineStoreDbContext>()
    .UseSqlServer("server=(local);database=OnlineStore;integrated security=yes;")
    .Options;

// Create DbContext instance
var dbContext = new OnlineStoreDbContext(options);

using (var repository = new SalesRepository(dbContext))
{
    // Retrieve entity by id instance for entity
    var entity = await repository.GetOrderHeaderAsync(new OrderHeader(1));
    
    // Set values for properties
    // e.g. entity.Total = 29.99m;
    
    // Update entity in database
    await repository.CommitChangesAsync();)
}
```

#### Delete

```csharp
// Create options for DbContext instance
var options = new DbContextOptionsBuilder<OnlineStoreDbContext>()
    .UseSqlServer("server=(local);database=OnlineStore;integrated security=yes;")
    .Options;

// Create DbContext instance
var dbContext = new OnlineStoreDbContext(options);

using (var repository = new SalesRepository(dbContext))
{
    // Retrieve entity by id instance for entity
    var entity = await repository.GetOrderHeaderAsync(new OrderHeader(1));
    
    // Remove entity in database
    repository.Remove(entity);

    await respository.CommitChangesAsync();
}
```

#### How works all code together?

We have created a *OnlineStoreDbContext* instance, that instance use the connection string from *DbContextOptionsBuilder* and inside of *OnModelCreating* method there is the configuration for all mappings, that's because it's more a stylish way to mapping entities instead of add a lot of lines inside of *OnModelCreating*.

Later, for example we create an instance of SalesRepository passing a valid instance of *OnlineStoreDbContext* and then we can access to repository's operations.

For this architecture implementation we are using the DotNet naming conventions: PascalCase for classes, interfaces and methods; camelCase for parameters.

Namespaces for generated code:

    EntityLayer
    DataLayer
    DataLayer\Contracts
    DataLayer\DataContracts
    DataLayer\Configurations
    DataLayer\Repositories

Inside of EntityLayer We'll place all entities, in this context entity means a class that represents a table or view from database, sometimes entity is named POCO (Plain Old Common language runtime Object) than means a class with only properties not methods nor other things (events).

Inside of DataLayer We'll place DbContext and AppSettings because they're common classes for DataLayer.

Inside of DataLayer\Contracts We'll place all interfaces that represent operations catalog, we're focusing on schemas and we'll create one interface per schema and Store contract for default schema (dbo).

Inside of DataLayer\DataContracts We'll place all object definitions for returned values from Contracts namespace, for now this directory would be empty.

Inside of DataLayer\Configurations We'll place all object definition related to mapping a class for database access.

Inside of DataLayer\Repositories We'll place the implementations for Contracts definitons.

Inside of EntityLayer and DataLayer\Mapping We'll create one directory per schema without include the default schema.

We can review the link about EF Core for enterprise, and we can understand this guide allow to us generate all of that code to reduce time in code writing.

## Code Review

We'll review some the output code for one entity to understand the design:

Code for *OrderHeader* class:

```csharp
using System;
using OnLineStore.Core.EntityLayer;
using OnLineStore.Core.EntityLayer.Sales;
using OnLineStore.Core.EntityLayer.HumanResources;
using System.Collections.ObjectModel;

namespace OnLineStore.Core.EntityLayer.Sales
{
	public partial class OrderHeader : IAuditEntity
	{
		public OrderHeader()
		{
		}

		public OrderHeader(long? orderHeaderID)
		{
			OrderHeaderID = orderHeaderID;
		}

		public long? OrderHeaderID { get; set; }

		public short? OrderStatusID { get; set; }

		public int? CustomerID { get; set; }

		public int? EmployeeID { get; set; }

		public int? ShipperID { get; set; }

		public DateTime? OrderDate { get; set; }

		public decimal? Total { get; set; }

		public short? CurrencyID { get; set; }

		public Guid? PaymentMethodID { get; set; }

		public int? DetailsCount { get; set; }

		public long? ReferenceOrderID { get; set; }

		public string Comments { get; set; }

		public string CreationUser { get; set; }

		public DateTime? CreationDateTime { get; set; }

		public string LastUpdateUser { get; set; }

		public DateTime? LastUpdateDateTime { get; set; }

		public byte[] Timestamp { get; set; }

		public OnLineStore.Core.EntityLayer.Currency CurrencyFk { get; set; }

		public OnLineStore.Core.EntityLayer.Sales.Customer CustomerFk { get; set; }

		public OnLineStore.Core.EntityLayer.HumanResources.Employee EmployeeFk { get; set; }

		public OnLineStore.Core.EntityLayer.Sales.OrderStatus OrderStatusFk { get; set; }

		public OnLineStore.Core.EntityLayer.Sales.PaymentMethod PaymentMethodFk { get; set; }

		public OnLineStore.Core.EntityLayer.Sales.Shipper ShipperFk { get; set; }

		public Collection<OrderDetail> OrderDetailList { get; set; }
	}
}
```

Code for *OnlineStoreDbContext* class:

```csharp
using System;
using Microsoft.EntityFrameworkCore;
using OnLineStore.Core.EntityLayer;
using OnLineStore.Core.DataLayer.Configurations;
using OnLineStore.Core.EntityLayer.HumanResources;
using OnLineStore.Core.EntityLayer.Sales;
using OnLineStore.Core.EntityLayer.Warehouse;
using OnLineStore.Core.DataLayer.Configurations.HumanResources;
using OnLineStore.Core.DataLayer.Configurations.Sales;
using OnLineStore.Core.DataLayer.Configurations.Warehouse;

namespace OnLineStore.Core.DataLayer
{
	public class OnLineStoreDbContext : DbContext
	{
		public OnLineStoreDbContext(DbContextOptions<OnLineStoreDbContext> options)
			: base(options)
		{
		}

		public DbSet<ChangeLog> ChangeLogs { get; set; }

		public DbSet<ChangeLogExclusion> ChangeLogExclusions { get; set; }

		public DbSet<Country> Countries { get; set; }

		public DbSet<CountryCurrency> CountryCurrencies { get; set; }

		public DbSet<Currency> Currencies { get; set; }

		public DbSet<EventLog> EventLogs { get; set; }

		public DbSet<Employee> Employees { get; set; }

		public DbSet<EmployeeAddress> EmployeeAddresses { get; set; }

		public DbSet<EmployeeEmail> EmployeeEmails { get; set; }

		public DbSet<Customer> Customers { get; set; }

		public DbSet<CustomerAddress> CustomerAddresses { get; set; }

		public DbSet<CustomerEmail> CustomerEmails { get; set; }

		public DbSet<OrderDetail> OrderDetails { get; set; }

		public DbSet<OrderHeader> OrderHeaders { get; set; }

		public DbSet<OrderStatus> OrderStatuses { get; set; }

		public DbSet<PaymentMethod> PaymentMethods { get; set; }

		public DbSet<Shipper> Shippers { get; set; }

		public DbSet<Location> Locations { get; set; }

		public DbSet<Product> Products { get; set; }

		public DbSet<ProductCategory> ProductCategories { get; set; }

		public DbSet<ProductInventory> ProductInventories { get; set; }

		public DbSet<EmployeeInfo> EmployeeInfos { get; set; }

		public DbSet<OrderSummary> OrderSummaries { get; set; }

		protected override void OnModelCreating(ModelBuilder modelBuilder)
		{
			// Apply all configurations for tables
			
			modelBuilder
				.ApplyConfiguration(new ChangeLogConfiguration())
				.ApplyConfiguration(new ChangeLogExclusionConfiguration())
				.ApplyConfiguration(new CountryConfiguration())
				.ApplyConfiguration(new CountryCurrencyConfiguration())
				.ApplyConfiguration(new CurrencyConfiguration())
				.ApplyConfiguration(new EventLogConfiguration())
				.ApplyConfiguration(new EmployeeConfiguration())
				.ApplyConfiguration(new EmployeeAddressConfiguration())
				.ApplyConfiguration(new EmployeeEmailConfiguration())
				.ApplyConfiguration(new CustomerConfiguration())
				.ApplyConfiguration(new CustomerAddressConfiguration())
				.ApplyConfiguration(new CustomerEmailConfiguration())
				.ApplyConfiguration(new OrderDetailConfiguration())
				.ApplyConfiguration(new OrderHeaderConfiguration())
				.ApplyConfiguration(new OrderStatusConfiguration())
				.ApplyConfiguration(new PaymentMethodConfiguration())
				.ApplyConfiguration(new ShipperConfiguration())
				.ApplyConfiguration(new LocationConfiguration())
				.ApplyConfiguration(new ProductConfiguration())
				.ApplyConfiguration(new ProductCategoryConfiguration())
				.ApplyConfiguration(new ProductInventoryConfiguration())
			;
			
			// Apply all configurations for views
			
			modelBuilder
				.ApplyConfiguration(new EmployeeInfoConfiguration())
				.ApplyConfiguration(new OrderSummaryConfiguration())
			;
			
			base.OnModelCreating(modelBuilder);
		}
	}
}
```

Code for *OrderHeaderConfiguration* class:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using OnLineStore.Core.EntityLayer.Sales;

namespace OnLineStore.Core.DataLayer.Configurations.Sales
{
	public class OrderHeaderConfiguration : IEntityTypeConfiguration<OnLineStore.Core.EntityLayer.Sales.OrderHeader>
	{
		public void Configure(EntityTypeBuilder<OnLineStore.Core.EntityLayer.Sales.OrderHeader> builder)
		{
			// Set configuration for entity
			builder.ToTable("OrderHeader", "Sales");
			
			// Set key for entity
			builder.HasKey(p => p.OrderHeaderID);
			
			// Set identity for entity (auto increment)
			builder.Property(p => p.OrderHeaderID).UseSqlServerIdentityColumn();
			
			// Set configuration for columns
			builder.Property(p => p.OrderHeaderID).HasColumnType("bigint").IsRequired();
			builder.Property(p => p.OrderStatusID).HasColumnType("smallint").IsRequired();
			builder.Property(p => p.CustomerID).HasColumnType("int").IsRequired();
			builder.Property(p => p.EmployeeID).HasColumnType("int");
			builder.Property(p => p.ShipperID).HasColumnType("int");
			builder.Property(p => p.OrderDate).HasColumnType("datetime").IsRequired();
			builder.Property(p => p.Total).HasColumnType("decimal(12, 4)").IsRequired();
			builder.Property(p => p.CurrencyID).HasColumnType("smallint");
			builder.Property(p => p.PaymentMethodID).HasColumnType("uniqueidentifier");
			builder.Property(p => p.DetailsCount).HasColumnType("int").IsRequired();
			builder.Property(p => p.ReferenceOrderID).HasColumnType("bigint");
			builder.Property(p => p.Comments).HasColumnType("varchar(max)");
			builder.Property(p => p.CreationUser).HasColumnType("varchar(25)").IsRequired();
			builder.Property(p => p.CreationDateTime).HasColumnType("datetime").IsRequired();
			builder.Property(p => p.LastUpdateUser).HasColumnType("varchar(25)");
			builder.Property(p => p.LastUpdateDateTime).HasColumnType("datetime");
			builder.Property(p => p.Timestamp).HasColumnType("timestamp(8)");
			
			// Set concurrency token for entity
			builder
				.Property(p => p.Timestamp)
				.ValueGeneratedOnAddOrUpdate()
				.IsConcurrencyToken();
			
			// Add configuration for foreign keys
			builder
				.HasOne(p => p.CurrencyFk)
				.WithMany(b => b.OrderHeaderList)
				.HasForeignKey(p => p.CurrencyID)
				.HasConstraintName("FK_Sales_OrderHeader_Currency");
			
			builder
				.HasOne(p => p.CustomerFk)
				.WithMany(b => b.OrderHeaderList)
				.HasForeignKey(p => p.CustomerID)
				.HasConstraintName("FK_Sales_OrderHeader_Customer");
			
			builder
				.HasOne(p => p.EmployeeFk)
				.WithMany(b => b.OrderHeaderList)
				.HasForeignKey(p => p.EmployeeID)
				.HasConstraintName("FK_Sales_OrderHeader_Employee");
			
			builder
				.HasOne(p => p.OrderStatusFk)
				.WithMany(b => b.OrderHeaderList)
				.HasForeignKey(p => p.OrderStatusID)
				.HasConstraintName("FK_Sales_OrderHeader_OrderStatus");
			
			builder
				.HasOne(p => p.PaymentMethodFk)
				.WithMany(b => b.OrderHeaderList)
				.HasForeignKey(p => p.PaymentMethodID)
				.HasConstraintName("FK_Sales_OrderHeader_PaymentMethod");
			
			builder
				.HasOne(p => p.ShipperFk)
				.WithMany(b => b.OrderHeaderList)
				.HasForeignKey(p => p.ShipperID)
				.HasConstraintName("FK_Sales_OrderHeader_Shipper");
			
		}
	}
}
```

Code for *ISalesRepository* interface:

```csharp
using System;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;
using OnLineStore.Core.EntityLayer;
using OnLineStore.Core.DataLayer.Contracts;
using OnLineStore.Core.EntityLayer.HumanResources;
using OnLineStore.Core.EntityLayer.Sales;
using OnLineStore.Core.EntityLayer.Warehouse;
using OnLineStore.Core.DataLayer.DataContracts;

namespace OnLineStore.Core.DataLayer.Contracts
{
	public interface ISalesRepository : IRepository
	{
		IQueryable<Customer> GetCustomers();

		Task<Customer> GetCustomerAsync(Customer entity);

		IQueryable<CustomerAddress> GetCustomerAddresses(int? customerID = null);

		Task<CustomerAddress> GetCustomerAddressAsync(CustomerAddress entity);

		IQueryable<CustomerEmail> GetCustomerEmails(int? customerID = null);

		Task<CustomerEmail> GetCustomerEmailAsync(CustomerEmail entity);

		IQueryable<OrderDetail> GetOrderDetails(long? orderHeaderID = null, int? productID = null);

		Task<OrderDetail> GetOrderDetailAsync(OrderDetail entity);

		Task<OrderDetail> GetOrderDetailByOrderHeaderIDAndProductIDAsync(OrderDetail entity);

		IQueryable<OrderHeaderDto> GetOrderHeaders(short? currencyID = null, int? customerID = null, int? employeeID = null, short? orderStatusID = null, Guid? paymentMethodID = null, int? shipperID = null);

		Task<OrderHeader> GetOrderHeaderAsync(OrderHeader entity);

		IQueryable<OrderStatus> GetOrderStatuses();

		Task<OrderStatus> GetOrderStatusAsync(OrderStatus entity);

		IQueryable<PaymentMethod> GetPaymentMethods();

		Task<PaymentMethod> GetPaymentMethodAsync(PaymentMethod entity);

		IQueryable<Shipper> GetShippers();

		Task<Shipper> GetShipperAsync(Shipper entity);

		IQueryable<OrderSummary> GetOrderSummaries();
	}
}
```

Code for *OrderDto* class:

```csharp
using System;

namespace OnLineStore.Core.DataLayer.DataContracts
{
	public class OrderHeaderDto
	{
		public Int64? OrderHeaderID { get; set; }

		public Int16? OrderStatusID { get; set; }

		public Int32? CustomerID { get; set; }

		public Int32? EmployeeID { get; set; }

		public Int32? ShipperID { get; set; }

		public DateTime? OrderDate { get; set; }

		public Decimal? Total { get; set; }

		public Int16? CurrencyID { get; set; }

		public Guid? PaymentMethodID { get; set; }

		public Int32? DetailsCount { get; set; }

		public Int64? ReferenceOrderID { get; set; }

		public String Comments { get; set; }

		public String CreationUser { get; set; }

		public DateTime? CreationDateTime { get; set; }

		public String LastUpdateUser { get; set; }

		public DateTime? LastUpdateDateTime { get; set; }

		public Byte[] Timestamp { get; set; }

		public String CurrencyCurrencyName { get; set; }

		public String CurrencyCurrencySymbol { get; set; }

		public String CustomerCompanyName { get; set; }

		public String CustomerContactName { get; set; }

		public String EmployeeFirstName { get; set; }

		public String EmployeeMiddleName { get; set; }

		public String EmployeeLastName { get; set; }

		public DateTime? EmployeeBirthDate { get; set; }

		public String OrderStatusDescription { get; set; }

		public String PaymentMethodPaymentMethodName { get; set; }

		public String PaymentMethodPaymentMethodDescription { get; set; }

		public String ShipperCompanyName { get; set; }

		public String ShipperContactName { get; set; }
	}
}
```

Code for *Repository* class:

```csharp
using System;
using System.Threading.Tasks;
using OnLineStore.Core.EntityLayer;

namespace OnLineStore.Core.DataLayer.Contracts
{
	public class Repository
	{
		protected bool Disposed;
		protected OnLineStoreDbContext DbContext;

		public Repository(OnLineStoreDbContext dbContext)
		{
			DbContext = dbContext;
		}

		public void Dispose()
		{
			if (!Disposed)
			{
				DbContext?.Dispose();
			
				Disposed = true;
			}
		}

		public virtual void Add<TEntity>(TEntity entity) where TEntity : class
		{
			// Cast entity to IAuditEntity
			if(entity is IAuditEntity cast)
			{
				// Set creation datetime
				cast.CreationDateTime = DateTime.Now;
			}
			
			DbContext.Add(entity);
		}

		public virtual void Update<TEntity>(TEntity entity) where TEntity : class
		{
			// Cast entity to IAuditEntity
			if (entity is IAuditEntity cast)
			{
				// Set update datetime
				cast.LastUpdateDateTime = DateTime.Now;
			}
			
			DbContext.Update(entity);
		}

		public virtual void Remove<TEntity>(TEntity entity) where TEntity : class
		{
			DbContext.Remove(entity);
		}

		public int CommitChanges()
			=> DbContext.SaveChanges();

		public Task<int> CommitChangesAsync()
			=> DbContext.SaveChangesAsync();
	}
}
```

Code for *SalesRepository* class:

```csharp
using System;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;
using OnLineStore.Core.EntityLayer;
using OnLineStore.Core.DataLayer.Contracts;
using OnLineStore.Core.EntityLayer.HumanResources;
using OnLineStore.Core.EntityLayer.Sales;
using OnLineStore.Core.EntityLayer.Warehouse;
using OnLineStore.Core.DataLayer.DataContracts;

namespace OnLineStore.Core.DataLayer.Repositories
{
	public class SalesRepository : Repository, ISalesRepository
	{
		public SalesRepository(OnLineStoreDbContext dbContext)
			: base(dbContext)
		{
		}

		public IQueryable<Customer> GetCustomers()
			=> DbContext.Customers;

		public async Task<Customer> GetCustomerAsync(Customer entity)
			=> await DbContext.Customers.FirstOrDefaultAsync(item => item.CustomerID == entity.CustomerID);

		public IQueryable<CustomerAddress> GetCustomerAddresses(int? customerID = null)
		{
			// Get query from DbSet
			var query = DbContext.CustomerAddresses.AsQueryable();
			
			// Filter by: 'CustomerID'
			if (customerID.HasValue)
				query = query.Where(item => item.CustomerID == customerID);
			
			return query;
		}

		public async Task<CustomerAddress> GetCustomerAddressAsync(CustomerAddress entity)
			=> await DbContext.CustomerAddresses.FirstOrDefaultAsync(item => item.CustomerAddressID == entity.CustomerAddressID);

		public IQueryable<CustomerEmail> GetCustomerEmails(int? customerID = null)
		{
			// Get query from DbSet
			var query = DbContext.CustomerEmails.AsQueryable();
			
			// Filter by: 'CustomerID'
			if (customerID.HasValue)
				query = query.Where(item => item.CustomerID == customerID);
			
			return query;
		}

		public async Task<CustomerEmail> GetCustomerEmailAsync(CustomerEmail entity)
			=> await DbContext.CustomerEmails.FirstOrDefaultAsync(item => item.CustomerEmailID == entity.CustomerEmailID);

		public IQueryable<OrderDetail> GetOrderDetails(long? orderHeaderID = null, int? productID = null)
		{
			// Get query from DbSet
			var query = DbContext.OrderDetails.AsQueryable();
			
			// Filter by: 'OrderHeaderID'
			if (orderHeaderID.HasValue)
				query = query.Where(item => item.OrderHeaderID == orderHeaderID);
			
			// Filter by: 'ProductID'
			if (productID.HasValue)
				query = query.Where(item => item.ProductID == productID);
			
			return query;
		}

		public async Task<OrderDetail> GetOrderDetailAsync(OrderDetail entity)
			=> await DbContext.OrderDetails.FirstOrDefaultAsync(item => item.OrderDetailID == entity.OrderDetailID);

		public async Task<OrderDetail> GetOrderDetailByOrderHeaderIDAndProductIDAsync(OrderDetail entity)
			=> await DbContext.OrderDetails.FirstOrDefaultAsync(item => item.OrderHeaderID == entity.OrderHeaderID && item.ProductID == entity.ProductID);

		public IQueryable<OrderHeaderDto> GetOrderHeaders(short? currencyID = null, int? customerID = null, int? employeeID = null, short? orderStatusID = null, Guid? paymentMethodID = null, int? shipperID = null)
		{
			// Get query from DbSet
			var query = from orderHeader in DbContext.Set<OrderHeader>()
				join currencyJoin in DbContext.Currencies on orderHeader.CurrencyID equals currencyJoin.CurrencyID into currencyTemp
					from currency in currencyTemp.DefaultIfEmpty()
				join customer in DbContext.Customers on orderHeader.CustomerID equals customer.CustomerID
				join employeeJoin in DbContext.Employees on orderHeader.EmployeeID equals employeeJoin.EmployeeID into employeeTemp
					from employee in employeeTemp.DefaultIfEmpty()
				join orderStatus in DbContext.OrderStatuses on orderHeader.OrderStatusID equals orderStatus.OrderStatusID
				join paymentMethodJoin in DbContext.PaymentMethods on orderHeader.PaymentMethodID equals paymentMethodJoin.PaymentMethodID into paymentMethodTemp
					from paymentMethod in paymentMethodTemp.DefaultIfEmpty()
				join shipperJoin in DbContext.Shippers on orderHeader.ShipperID equals shipperJoin.ShipperID into shipperTemp
					from shipper in shipperTemp.DefaultIfEmpty()
				select new OrderHeaderDto
				{
					OrderHeaderID = orderHeader.OrderHeaderID,
					OrderStatusID = orderHeader.OrderStatusID,
					CustomerID = orderHeader.CustomerID,
					EmployeeID = orderHeader.EmployeeID,
					ShipperID = orderHeader.ShipperID,
					OrderDate = orderHeader.OrderDate,
					Total = orderHeader.Total,
					CurrencyID = orderHeader.CurrencyID,
					PaymentMethodID = orderHeader.PaymentMethodID,
					DetailsCount = orderHeader.DetailsCount,
					ReferenceOrderID = orderHeader.ReferenceOrderID,
					Comments = orderHeader.Comments,
					CreationUser = orderHeader.CreationUser,
					CreationDateTime = orderHeader.CreationDateTime,
					LastUpdateUser = orderHeader.LastUpdateUser,
					LastUpdateDateTime = orderHeader.LastUpdateDateTime,
					Timestamp = orderHeader.Timestamp,
					CurrencyCurrencyName = currency == null ? string.Empty : currency.CurrencyName,
					CurrencyCurrencySymbol = currency == null ? string.Empty : currency.CurrencySymbol,
					CustomerCompanyName = customer == null ? string.Empty : customer.CompanyName,
					CustomerContactName = customer == null ? string.Empty : customer.ContactName,
					EmployeeFirstName = employee == null ? string.Empty : employee.FirstName,
					EmployeeMiddleName = employee == null ? string.Empty : employee.MiddleName,
					EmployeeLastName = employee == null ? string.Empty : employee.LastName,
					EmployeeBirthDate = employee == null ? default(DateTime?) : employee.BirthDate,
					OrderStatusDescription = orderStatus == null ? string.Empty : orderStatus.Description,
					PaymentMethodPaymentMethodName = paymentMethod == null ? string.Empty : paymentMethod.PaymentMethodName,
					PaymentMethodPaymentMethodDescription = paymentMethod == null ? string.Empty : paymentMethod.PaymentMethodDescription,
					ShipperCompanyName = shipper == null ? string.Empty : shipper.CompanyName,
					ShipperContactName = shipper == null ? string.Empty : shipper.ContactName,
				};
			
			// Filter by: 'CurrencyID'
			if (currencyID.HasValue)
				query = query.Where(item => item.CurrencyID == currencyID);
			
			// Filter by: 'CustomerID'
			if (customerID.HasValue)
				query = query.Where(item => item.CustomerID == customerID);
			
			// Filter by: 'EmployeeID'
			if (employeeID.HasValue)
				query = query.Where(item => item.EmployeeID == employeeID);
			
			// Filter by: 'OrderStatusID'
			if (orderStatusID.HasValue)
				query = query.Where(item => item.OrderStatusID == orderStatusID);
			
			// Filter by: 'PaymentMethodID'
			if (paymentMethodID != null)
				query = query.Where(item => item.PaymentMethodID == paymentMethodID);
			
			// Filter by: 'ShipperID'
			if (shipperID.HasValue)
				query = query.Where(item => item.ShipperID == shipperID);
			
			return query;
		}

		public async Task<OrderHeader> GetOrderHeaderAsync(OrderHeader entity)
		{
			return await DbContext.OrderHeaders
				.Include(p => p.CurrencyFk)
				.Include(p => p.CustomerFk)
				.Include(p => p.EmployeeFk)
				.Include(p => p.OrderStatusFk)
				.Include(p => p.PaymentMethodFk)
				.Include(p => p.ShipperFk)
				.FirstOrDefaultAsync(item => item.OrderHeaderID == entity.OrderHeaderID);
		}

		public IQueryable<OrderStatus> GetOrderStatuses()
			=> DbContext.OrderStatuses;

		public async Task<OrderStatus> GetOrderStatusAsync(OrderStatus entity)
			=> await DbContext.OrderStatuses.FirstOrDefaultAsync(item => item.OrderStatusID == entity.OrderStatusID);

		public IQueryable<PaymentMethod> GetPaymentMethods()
			=> DbContext.PaymentMethods;

		public async Task<PaymentMethod> GetPaymentMethodAsync(PaymentMethod entity)
			=> await DbContext.PaymentMethods.FirstOrDefaultAsync(item => item.PaymentMethodID == entity.PaymentMethodID);

		public IQueryable<Shipper> GetShippers()
			=> DbContext.Shippers;

		public async Task<Shipper> GetShipperAsync(Shipper entity)
			=> await DbContext.Shippers.FirstOrDefaultAsync(item => item.ShipperID == entity.ShipperID);

		public IQueryable<OrderSummary> GetOrderSummaries()
			=> DbContext.OrderSummaries;
	}
}
```

## Points of Interest

* **CatFactory** doesn't have command line for Nuget because from my point of view it will be a big trouble to allow set values for all settings because we have a lot of settings for EntityFrameworkCoreProjectSettings, I think at this moment is more simple to create a console project to generate the code and then developer move generated files for existing project and make a code refactor if applies
* **CatFactory** doesn't have UI now because at the beginning of this project .NET Core had no an standard UI, but we're working on UI for **CatFactory** maybe we'll choose Angular =^^=
* Now we are focused on Entity Framework Core and *Dapper* but in future there will be Web API, Unit Tests and other things :)
* **CatFactory** has a package for *Dapper*, at this moment there isn't article for that but the way to use is similar for *Entity Framework Core*; you can install **CatFactory.Dapper** package from Nuget
* We're working in continuous updates to provide better help for developers

## Related Links

[`Entity Framework Core 2 for the Enterprise`](https://www.codeproject.com/Articles/1160586/Entity-Framework-Core-2-for-the-Enterprise)

[`Creating Web API in ASP.NET Core 2.0`](https://www.codeproject.com/Articles/1264219/Creating-Web-API-in-ASP-NET-Core-2-0)

## Code Improvements

    Add author's information for output files

### Bugs?

If you get any exception with CatFactory packages, please use these links:

[`Issue tracker`](https://github.com/hherzl/CatFactory.SqlServer/issues) for CatFactory.SqlServer in GitHub

[`Issue tracker`](https://github.com/hherzl/CatFactory.EntityFrameworkCore/issues) for CatFactory.EntityFrameworkCore in GitHub
