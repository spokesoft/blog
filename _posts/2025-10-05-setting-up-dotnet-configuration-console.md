---
title: "Setting up .NET Configuration in Console Applications"
description: "A guide to configuring .NET console applications using the built-in configuration framework."
date: 2025-10-05 10:00:00
categories: [dotnet, configuration]
tags: [dotnet, configuration, console]
author: jtrumbull
---

## Dependencies

To get started with .NET configuration in a console application, you'll need to add one or more [configuration providers](https://learn.microsoft.com/en-us/dotnet/core/extensions/configuration-providers) to your project. You can do this by adding the following NuGet packages to your project:

```bash
dotnet add package Microsoft.Extensions.Configuration.Json
dotnet add package Microsoft.Extensions.Configuration.EnvironmentVariables
dotnet add package Microsoft.Extensions.Configuration.CommandLine
```

> Unless you need a custom configuration provider, or want explicit dependencies, you don't need to add `Microsoft.Extensions.Configuration` directly.
{: .prompt-tip }

## Setting up Configuration

In your `Program.cs` or main application file, you can set up the configuration builder to read from various sources. Here's an example of how to configure a console application to read from a JSON file, environment variables, and command-line arguments:

```csharp
using Microsoft.Extensions.Configuration;

var configuration = new ConfigurationBuilder()
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: false)
    .AddEnvironmentVariables(prefix: "Spokesoft_")
    .AddCommandLine(args)
    .Build();

var connectionString = configuration.GetConnectionString("DefaultConnection");
Console.WriteLine($"Connection String: {connectionString}");
```

> The `reloadOnChange` feature is designed for long-running applications like web servers, which can monitor for file updates. Since a typical console application starts, performs its task, and then immediately terminates, there is no persistent process left to detect any changes. Each time you run the CLI, it naturally reads the latest configuration from the files, making the concept of an automatic reload redundant.
{: .prompt-info }

### Order of Precedence

The order in which you add configuration providers matters. In the example above, command-line arguments will override environment variables, which in turn will override settings from the JSON file. This allows for flexible configuration management.

### App Settings

Create an `appsettings.json` file in the root of your project to store configuration settings:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;"
  }
}
```

After creating the file, ensure it is copied to the output directory by adding the following to your `.csproj` file:

```xml
  <ItemGroup>
    <None Update="appsettings.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
  </ItemGroup>
```

#### Trying it out

Now, when you run your console application, it will read the connection string from `appsettings.json` by default. You can override it using environment variables or command-line arguments.

```powershell
dotnet run 
# Connection String: Server=localhost;
```

### Environment Variables

You can also use environment variables to configure your application. This is particularly useful for sensitive information like connection strings or API keys. To set an environment variable, you can use the following command in your terminal:

```powershell
$env:Spokesoft_ConnectionStrings__DefaultConnection = "Server=a.host;"
dotnet run
# Connection String: Server=a.host;
```

### Command-Line Arguments

Finally, you can override configuration settings using command-line arguments when running your application:

```powershell
$env:Spokesoft_ConnectionStrings__DefaultConnection = "Server=a.host;"
dotnet run --ConnectionStrings:DefaultConnection="Server=b.host;"
# Connection String: Server=b.host;
```

This setup provides a flexible and powerful way to manage configuration in your .NET console applications, allowing you to easily switch between different environments and settings as needed.
