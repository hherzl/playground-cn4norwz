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
|**CatFactory.AspNetCore**|1.0.0-beta-sun-build37|Provides scaffold for Entity Framework Core|

#### Install with Nuget UI

	Right Click on Project
	Select Manage Nuget Packages
	Select Browse tab
    Search CatFactory.AspNetCore
    Install

#### Install with Package Manager Console

	Go to Tools menu
	Select Nuget Package Manager
	Select Package Manager Console
    Run the following command: **Install-Package CatFactory.EntityFrameworkCore -Version 1.0.0-beta-sun-build35**

Save all changes and build the project.

### Step 03 - Add Code to scaffold

Once We have the packages installed on our project, We can add the following code in *Main* method:

```csharp
// Import database
var database = SqlServerDatabaseFactory
    .Import("server=(local);database=OnlineStore;integrated security=yes;", "dbo.sysdiagrams");

// Create instance of Entity Framework Core Project
var entityFrameworkProject = new EntityFrameworkCoreProject
{
    Name = "OnlineStore.Core",
    Database = database,
    OutputDirectory = @"C:\Projects\OnlineStore.Core"
};

// Apply settings for project
entityFrameworkProject.GlobalSelection(settings =>
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

entityFrameworkProject.Selection("Sales.OrderHeader", settings => settings.EntitiesWithDataContracts = true);

// Build features for project, group all entities by schema into a feature
entityFrameworkProject.BuildFeatures();

// Scaffolding =^^=
entityFrameworkProject
    .ScaffoldEntityLayer()
    .ScaffoldDataLayer();

var aspNetCoreProject = entityFrameworkProject
    .CreateAspNetCoreProject("OnlineStore.WebAPI", @"C:\Temp\CatFactory.AspNetCore\OnlineStore.WebAPI");

aspNetCoreProject.GlobalSelection(settings => settings.ForceOverwrite = true);

aspNetCoreProject.Selection("Sales.OrderDetail", settings =>
{
    settings
        .RemoveAction<ReadAllAction>()
        .RemoveAction<ReadByKeyAction>()
        .RemoveAction<AddEntityAction>()
        .RemoveAction<UpdateEntityAction>()
        .RemoveAction<RemoveEntityAction>();
});

// Scaffolding ==^^==
aspNetCoreProject.ScaffoldAspNetCore();
```

### Step 04 - Create Web API Project

The following code shows how to use the scaffolded code, keep on mind We'll need to set up dependency injection and anothers things, please check in related links for more information.

## Code Review

We'll review some the output code for one entity to understand the design:

#### Controller

```csharp
using System;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;
using OnlineStore.Core.DataLayer.Contracts;
using OnlineStore.Core.DataLayer.Repositories;
using OnlineStore.WebAPI.Responses;
using OnlineStore.WebAPI.Requests;
using OnlineStore.Core.EntityLayer.Sales;
using OnlineStore.Core.DataLayer.DataContracts;

namespace OnlineStore.WebAPI.Controllers
{
	[Route("api/[controller]")]
	[ApiController]
	public class SalesController : ControllerBase
	{
		protected readonly ISalesRepository Repository;
		protected ILogger Logger;

		public SalesController(ISalesRepository repository, ILogger<SalesController> logger)
		{
			Repository = repository;
			Logger = logger;
		}

		// Another actions

		[HttpGet("OrderHeader")]
		public async Task<IActionResult> GetOrderHeadersAsync(int? pageSize = 10, int? pageNumber = 1, string currencyID = null, int? customerID = null, int? employeeID = null, short? orderStatusID = null, Guid? paymentMethodID = null, int? shipperID = null)
		{
			Logger?.LogDebug("'{0}' has been invoked", nameof(GetOrderHeadersAsync));
			
			var response = new PagedResponse<OrderHeaderDto>();
			
			try
			{
				// Get query from repository
				var query = Repository.GetOrderHeaders(currencyID, customerID, employeeID, orderStatusID, paymentMethodID, shipperID);
			
				// Set paging's information
				response.PageSize = (int)pageSize;
				response.PageNumber = (int)pageNumber;
				response.ItemsCount = await query.CountAsync();
			
				// Retrieve items by page size and page number, set model for response
				response.Model = await query.Paging(response.PageSize, response.PageNumber).ToListAsync();
			
				Logger?.LogInformation("Page {0} of {1}, Total of rows: {2}.", response.PageNumber, response.PageCount, response.ItemsCount);
			}
			catch (Exception ex)
			{
				response.SetError(Logger, nameof(GetOrderHeadersAsync), ex);
			}
			
			return response.ToHttpResponse();
		}

		[HttpGet("OrderHeader/{id}")]
		public async Task<IActionResult> GetOrderHeaderAsync(long? id)
		{
			Logger?.LogDebug("'{0}' has been invoked", nameof(GetOrderHeaderAsync));
			
			var response = new SingleResponse<OrderHeaderRequest>();
			
			try
			{
				// Retrieve entity by id
				var entity = await Repository.GetOrderHeaderAsync(new OrderHeader(id));
			
				if (entity != null)
				{
					response.Model = entity.ToRequest();
			
					Logger?.LogInformation("The entity was retrieved successfully");
				}
			}
			catch (Exception ex)
			{
				response.SetError(Logger, nameof(GetOrderHeaderAsync), ex);
			}
			
			return response.ToHttpResponse();
		}

		[HttpPost("OrderHeader")]
		public async Task<IActionResult> PostOrderHeaderAsync([FromBody]OrderHeaderRequest request)
		{
			Logger?.LogDebug("'{0}' has been invoked", nameof(PostOrderHeaderAsync));
			
			// Validate request model
			if (!ModelState.IsValid)
				return BadRequest(request);
			
			var response = new SingleResponse<OrderHeaderRequest>();
			
			try
			{
				var entity = request.ToEntity();
			
				// Add entity to database
				Repository.Add(entity);
			
				await Repository.CommitChangesAsync();
			
				response.Model = entity.ToRequest();
			
				Logger?.LogInformation("The entity was created successfully");
			}
			catch (Exception ex)
			{
				response.SetError(Logger, nameof(PostOrderHeaderAsync), ex);
			}
			
			return response.ToHttpResponse();
		}

		[HttpPut("OrderHeader/{id}")]
		public async Task<IActionResult> PutOrderHeaderAsync(long? id, [FromBody]OrderHeaderRequest request)
		{
			Logger?.LogDebug("'{0}' has been invoked", nameof(PutOrderHeaderAsync));
			
			// Validate request model
			if (!ModelState.IsValid)
				return BadRequest(request);
			
			var response = new Response();
			
			try
			{
				// Retrieve entity by id
				var entity = await Repository.GetOrderHeaderAsync(new OrderHeader(id));
			
				if (entity != null)
				{
					// todo:  Check properties to update
			
					// Apply changes on entity
					entity.OrderStatusID = request.OrderStatusID;
					entity.CustomerID = request.CustomerID;
					entity.EmployeeID = request.EmployeeID;
					entity.ShipperID = request.ShipperID;
					entity.OrderDate = request.OrderDate;
					entity.Total = request.Total;
					entity.CurrencyID = request.CurrencyID;
					entity.PaymentMethodID = request.PaymentMethodID;
					entity.DetailsCount = request.DetailsCount;
					entity.ReferenceOrderID = request.ReferenceOrderID;
					entity.Comments = request.Comments;
			
					// Save changes for entity in database
					Repository.Update(entity);
			
					await Repository.CommitChangesAsync();
			
					Logger?.LogInformation("The entity was updated successfully");
				}
			}
			catch (Exception ex)
			{
				response.SetError(Logger, nameof(PutOrderHeaderAsync), ex);
			}
			
			return response.ToHttpResponse();
		}

		[HttpDelete("OrderHeader/{id}")]
		public async Task<IActionResult> DeleteOrderHeaderAsync(long? id)
		{
			Logger?.LogDebug("'{0}' has been invoked", nameof(DeleteOrderHeaderAsync));
			
			var response = new Response();
			
			try
			{
				// Retrieve entity by id
				var entity = await Repository.GetOrderHeaderAsync(new OrderHeader(id));
			
				if (entity != null)
				{
					// Remove entity from database
					Repository.Remove(entity);
			
					await Repository.CommitChangesAsync();
			
					Logger?.LogInformation("The entity was deleted successfully");
				}
			}
			catch (Exception ex)
			{
				response.SetError(Logger, nameof(DeleteOrderHeaderAsync), ex);
			}
			
			return response.ToHttpResponse();
		}

		// Another actions
	}
}
```

#### Requests

```csharp
using System;
using System.ComponentModel.DataAnnotations;
using OnlineStore.Core.EntityLayer.Sales;

namespace OnlineStore.WebAPI.Requests
{
	public class OrderHeaderRequest
	{
		[Key]
		public long? OrderHeaderID { get; set; }

		[Required]
		public short? OrderStatusID { get; set; }

		[Required]
		public int? CustomerID { get; set; }

		public int? EmployeeID { get; set; }

		public int? ShipperID { get; set; }

		[Required]
		public DateTime? OrderDate { get; set; }

		[Required]
		public decimal? Total { get; set; }

		[StringLength(10)]
		public string CurrencyID { get; set; }

		public Guid? PaymentMethodID { get; set; }

		[Required]
		public int? DetailsCount { get; set; }

		public long? ReferenceOrderID { get; set; }

		public string Comments { get; set; }

		[Required]
		[StringLength(25)]
		public string CreationUser { get; set; }

		[Required]
		public DateTime? CreationDateTime { get; set; }

		[StringLength(25)]
		public string LastUpdateUser { get; set; }

		public DateTime? LastUpdateDateTime { get; set; }

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

[`Issue tracker`](https://github.com/hherzl/CatFactory.AspNetCore/issues) for CatFactory.AspNetCore in GitHub
