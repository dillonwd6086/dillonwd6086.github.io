---
layout: post
title:  "Dotnet Core SPROC Validation"
date:   2019-05-12 11:59:11 -0400
categories: jekyll update
---

## Goals

Create a structured means to specify what Stored Procedures (SPROCS) are required for a program to work
properly.  Once we have defined the required SPROCS then at start up create an application that has
the ability to verify in the database that those objects exist.

### Requirements 

- DotnetCore 2.1
- MySql Database

### Step 1 - Creating Test Procedures

Before I can create any thing that validates stored procedure existence, I need to create
a stored procedures in my database.  Here is the procedure I will use for testing.

```MySql
CREATE PROCEDURE Test_LoadData
  (IN id INT)
BEGIN
  SELECT * from Garbage where Garbage.Id = id;
END
```

### Step 2 - Creating Test Project

Now that we have an SPROC lets create the test dotnet core project and get that set up.

1.  Run `dotnet new console`

2.  Install the MySql Connector Libs `dotnet add package MySql.Data.EntityFrameworkCore`

### Step 3 - Writing the Validator

This will be the most involved part of this post

#### MySqlSpObject.cs

This object is meant to represent a MySql Stored Procedure and will be really lightweight just
mapping the name to a Database

```csharp
namespace sproc_validator
{
    public class MySqlSpObject
    {
        public string Db { get; set; }
        public string Name { get; set; }
    }
}
```

#### MySqlObjectValidator.cs

This is where the meat and potatoes for this program exists.  This handles the logic of
loading all of the SP names from the database and exposing the validation.  

We will create a validator that takes in the DbSecrets in the constructor which will 
be used to build up a connection string, since we do not want to hard code any credentials.
More information on how to safely store your credentials can be found at this [Post](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets?view=aspnetcore-2.2&tabs=linux).

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.Extensions.Options;
using MySql.Data.MySqlClient;

namespace sproc_validator 
{
    public class MySqlObjectValidator
    {
        public MySqlObjectValidator(DbSecrets secrets, string dbName = "")
        {
            _connectionString = secrets.GenerateConnectionString();
            _dbName = dbName;
            _loadedSps = new List<MySqlSpObject>();
            LoadAllStoredProcedures();
        }

        public void LoadAllStoredProcedures()
        {
            MySqlConnection conn = new MySqlConnection(_connectionString);

            try
            {
                conn.Open();

                var command = CreateListStoredProcedures();
                command.Connection = conn;

                var resultReader = command.ExecuteReader();

                while (resultReader.Read())
                {
                    _loadedSps.Add(new MySqlSpObject
                    {
                        Db = resultReader.GetString(0),
                        Name = resultReader.GetString(1)
                    });
                }
            }
            catch (MySqlException excp)
            {
                Console.WriteLine($"MySql call failed :: {excp.Message}");
            }
            finally
            {
                conn.Close();
            }
        }

        public bool DoesExist(string spName)
        {
            return _loadedSps.Any(_ => _.Name == spName);
        }

        private MySqlCommand CreateListStoredProcedures()
        {
            string spText = "SHOW PROCEDURE STATUS";

            if (_dbName != string.Empty)
            {
                spText += $" WHERE Db = '{_dbName}'";
            }
		}

		private readonly string _connectionString;
		private string _dbName;
		private List<MySqlSpObject> _loadedSps;
	}
}
```

#### DbSecrets.cs

This object holds the information on how to connect to our db, we will load this from our
appsettings.json.

```csharp
namespace sproc_validator
{
    public class DbSecrets
    {
        public string Password { get; set; }
        public string Hostname { get; set; }
        public string ConfiguredString { get; set; }

        public string GenerateConnectionString()
        {
            return ConfiguredString.Replace("{password}", password ?? "isnull")
                                   .Replace("{hostname}", hostname ?? "isnull");
        }
    }
}
```

#### StoredProcedures.cs

We will declare all of the stored procedures that we want to use in our program here.  I have found
that it is easiest to declare the name's of all procedures in a single static class for the project
that uses them.  This makes calling and updating there names easier.  This will also allow us to
reflect over all of the procedures used and validate that they exist in the database.

```csharp
namespace sproc_validator
{
    public static class StoredProcedures
    {
        public const string TestLoadData = "Test_LoadData";
    }
}
```

#### ValidatorExtensions.cs

Here are a few extension methods that will make writing this program easier and more expressive.

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Reflection;

namespace sproc_validator
{
    public static class ValidatorExtensions
    {
        public static List<T> GetAllPublicConstantValues<T>(this Type type)
        {
            return type
                .GetFields(
                    BindingFlags.Public | BindingFlags.Static | BindingFlags.FlattenHierarchy)
                .Where(fi => fi.IsLiteral && !fi.IsInitOnly && fi.FieldType == typeof(T))
                .Select(x => (T) x.GetRawConstantValue())
                .ToList();
        }

        public static string GetSpName(this string text)
        {
            var spCall = text.Split(' ')[1];
            return spCall;
        }
    }
}
```

#### Program.cs

Now lets right the main program that will run and validate all of stored procedures.

```csharp
using System;
using System.IO;
using System.Linq;
using Microsoft.Extensions.Configuration;

namespace sproc_validator
{
    class Program
    {
        static void main(string [] args)
        {
            var config = new ConfigurationBuilder()
                .SetBasePath(Directory.GetCurrentDirectory())
                .AddJsonFile("appsettings.json")
                .AddUserSecrets<DbSecrets>()
                .Build();

            var sqlValidator = new MySqlObjectValidator(new DbSecrets
            {
                Password = configuration["Db:Password"],
                Hostname = configuration["Db:Hostname"],
                ConfiguaredString = configuaration["ConnectionString"]
            }, dbName: "test");

            var paramsToValidate = typeof(StoredProcedures)
                                    .GetAllPublicConstantValues<string>();

            foreach(var param in paramsToValidate)
            {
                if(!sqlValidator.DoesExist(param.GetSpName()))
                {
                    Console.WriteLine($"ERROR!: {param.GetSpName()} Does not exist in the db!"); 
                }
                else
                {
                    Console.WriteLIne($"{param.GetSpName()} is VALID");
                }
            }
        }
    }
}
```

### Results

Once you run this you should get

![Result](/assets/sp-validator.png)
