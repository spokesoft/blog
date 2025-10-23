---
title: "Initializing .NET projects with clean architecture"
description: "Learn how to set up .NET projects using clean architecture principles."
date: 2025-10-04 10:00:00
categories: [dotnet, architecture]
tags: [dotnet, clean-architecture, project-setup]
---

# Creating the solution

To create a new .NET solution, start by opening your terminal and running the following command:

```bash
dotnet new sln -n MyApp
```

# Create the projects

Next, create the individual projects that will make up your solution. For a clean architecture, you typically need at least three projects: one for the domain, one for the application logic, and one for the infrastructure.

```bash
dotnet new classlib -o ./src/Domain
dotnet new classlib -o ./src/Application
dotnet new classlib -o ./src/Infrastructure
dotnet new webapi -o ./src/Api
```

# Add projects to the solution

Next, add the newly created projects to your solution using the following commands:

```bash
dotnet sln add ./src/Domain/Domain.csproj
dotnet sln add ./src/Application/Application.csproj
dotnet sln add ./src/Infrastructure/Infrastructure.csproj
dotnet sln add ./src/Api/Api.csproj
```

# Add project references

To establish the relationships between the projects, you need to add project references. This can be done using the following commands:

```bash
dotnet add ./src/Application/Application.csproj reference ./src/Domain/Domain.csproj
dotnet add ./src/Infrastructure/Infrastructure.csproj reference ./src/Application/Application.csproj
dotnet add ./src/Api/Api.csproj reference ./src/Infrastructure/Infrastructure.csproj
```

# All together now

Your terminal commands should look like this when put together:

```bash
dotnet new sln -n MyApp
dotnet new classlib -o ./src/Domain
dotnet new classlib -o ./src/Application
dotnet new classlib -o ./src/Infrastructure
dotnet new webapi -o ./src/Api
dotnet sln add ./src/Domain/Domain.csproj
dotnet sln add ./src/Application/Application.csproj
dotnet sln add ./src/Infrastructure/Infrastructure.csproj
dotnet sln add ./src/Api/Api.csproj
dotnet add ./src/Application/Application.csproj reference ./src/Domain/Domain.csproj
dotnet add ./src/Infrastructure/Infrastructure.csproj reference ./src/Application/Application.csproj
dotnet add ./src/Api/Api.csproj reference ./src/Infrastructure/Infrastructure.csproj
```

