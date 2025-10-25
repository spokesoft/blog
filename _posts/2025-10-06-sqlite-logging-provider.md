---
title: "SQLite Logging Provider"
description: "A guide to setting up SQLite logging in .NET applications."
date: 2025-10-06 10:00:00
categories: [dotnet, logging, sqlite]
tags: [dotnet, logging, sqlite]
author: jtrumbull
---

This post provides a step-by-step guide on how to implement SQLite as a logging provider in .NET applications using the `Microsoft.Extensions.Logging` framework. By the end of this tutorial, you'll be able to log application events directly into an SQLite database for easy querying and analysis.

The source code for this tutorial is available on [GitHub](https://github.com/jtrumbull/dotnet-logging-sqlite).

## Dependencies

To get started with SQLite logging in .NET, you'll need to add the following NuGet packages to your project:

```bash
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.Extensions.Configuration.Binder
dotnet add package Microsoft.Extensions.Configuration.Json
dotnet add package Microsoft.Extensions.Logging
```

## The Data Model

Define the data model that represents a log entry in the SQLite database.

### The Log Entry Model

Define a model to represent a log entry.

```csharp
using Microsoft.Extensions.Logging;

namespace DatabaseLogging;

public class LogEntry
{
    public long Id { get; set; }
    public EventId EventId { get; set; }
    public LogLevel Level { get; set; }
    public required string Category { get; set; }
    public required string Message { get; set; }
    public string? Exception { get; set; }
    public DateTime Timestamp { get; set; }
}
```

### Type configuration

Create an Entity Framework Core configuration class to map the `LogEntry` model to the SQLite database schema.

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Microsoft.EntityFrameworkCore.Storage.ValueConversion;
using Microsoft.Extensions.Logging;

namespace DatabaseLogging;

public class LogEntryTypeConfiguration : IEntityTypeConfiguration<LogEntry>
{
    // Configures the LogEntry entity for EF Core.
    public void Configure(EntityTypeBuilder<LogEntry> builder)
    {
        builder.ToTable("logs");
        builder.HasKey(e => e.Id);
        builder.Property(e => e.EventId)
            .HasConversion<EventIdConverter>()
            .IsRequired();
        builder.Property(e => e.Level)
            .HasConversion<LogLevelConverter>()
            .IsRequired();
        builder.Property(e => e.Category)
            .HasMaxLength(200)
            .IsRequired();
        builder.Property(e => e.Message)
            .HasMaxLength(1000)
            .IsRequired();
        builder.Property(e => e.Exception)
            .HasMaxLength(2000);
        builder.Property(e => e.Timestamp).IsRequired();
    }

    // Converts EventId to and from a string representation for database storage.
    private class EventIdConverter : ValueConverter<EventId, string>
    {
        public EventIdConverter() : base(
            v => ToProvider(v),
            v => FromProvider(v))
        { }

        public static string ToProvider(EventId eventId) => $"{eventId.Id}:{eventId.Name}";

        public static EventId FromProvider(string value)
        {
            var parts = value.Split(':', 2);
            if (parts.Length == 2 && int.TryParse(parts[0], out var id))
            {
                return new EventId(id, parts[1]);
            }
            return new EventId(0, value);
        }
    }

    // Converts LogLevel to and from a string representation for database storage.
    private class LogLevelConverter : ValueConverter<LogLevel, string>
    {
        public LogLevelConverter() : base(
            v => ToProvider(v),
            v => FromProvider(v)) {}

        public static string ToProvider(LogLevel level) => level.ToString();

        public static LogLevel FromProvider(string value)
            => Enum.TryParse<LogLevel>(value, out var level) ? level : LogLevel.None;        
    }
}
```

### Database Context

Create a `DbContext` to manage the SQLite database connection and the `LogEntry` entities.

```csharp
using Microsoft.EntityFrameworkCore;

namespace DatabaseLogging;

// DbContext for logging database operations
public class LoggingDbContext(
    DbContextOptions<LoggingDbContext> options) : DbContext(options)
{
    public DbSet<LogEntry> Logs { get; set; } = null!;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfiguration(new LogEntryTypeConfiguration());
    }
}
```

### Design Time Context Factory

Create a design-time factory for the `LoggingDbContext` to facilitate migrations and other design-time operations.

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Design;

namespace DatabaseLogging;

// Factory for creating LoggingDbContext instances at design time.
public class LoggingDbContextFactory : IDesignTimeDbContextFactory<LoggingDbContext>
{
    public LoggingDbContext CreateDbContext(string[] args)
    {
        var optionsBuilder = new DbContextOptionsBuilder<LoggingDbContext>();
        optionsBuilder.UseSqlite("Data Source=any.db");

        return new LoggingDbContext(optionsBuilder.Options);
    }
}
```

### Create the Migration

Create the initial migration to set up the database schema.

```bash
dotnet ef migrations add InitialCreate
```

## The Logger

Create a custom logger class that implements the [ILogger](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.ilogger) interface. This class will write log entries to a [channel](https://learn.microsoft.com/en-us/dotnet/core/extensions/channels).

```csharp
using System.Threading.Channels;
using Microsoft.Extensions.Logging;

namespace DatabaseLogging;

public class DatabaseLogger(
    string category,
    ChannelWriter<LogEntry> writer,
    LogLevel minLevel = LogLevel.Trace) : ILogger
{
    private readonly string _category = category;
    private readonly ChannelWriter<LogEntry> _writer = writer;
    private readonly LogLevel _minLevel = minLevel;

    public IDisposable? BeginScope<TState>(TState state) where TState : notnull
      => null;

    public bool IsEnabled(LogLevel level) 
      => level >= _minLevel;

    public void Log<TState>(
      LogLevel level, 
      EventId eventId, 
      TState state, 
      Exception? exception, 
      Func<TState, Exception?, string> formatter)
    {
        if (!IsEnabled(level))
            return;

        var message = formatter?.Invoke(state, exception)
            ?? state?.ToString()
            ?? string.Empty;

        var entry = new LogEntry
        {
            EventId = eventId,
            Level = level,
            Category = _category,
            Message = message,
            Exception = exception?.ToString(),
            Timestamp = DateTime.UtcNow
        };

        _writer.TryWrite(entry);
    }
}
```

## The Logger Provider

Create a logger provider that implements the [ILoggerProvider](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.iloggerprovider) interface. This class will manage the creation of `DatabaseLogger` instances.

```csharp
using System.Collections.Concurrent;
using System.Threading.Channels;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace DatabaseLogging;

public class DatabaseLoggerProvider(
    Channel<LogEntry> channel,
    IOptions<LoggingOptions> options) : ILoggerProvider
{
    private readonly ConcurrentDictionary<string, DatabaseLogger> _loggers = [];
    private readonly Dictionary<string, LogLevel> _categoryLevels = options.Value.LogLevel;
    private readonly ChannelWriter<LogEntry> _writer = channel.Writer;
    
    // Creates or retrieves a logger for the specified category.
    public ILogger CreateLogger(string category)
    {
        var minLevel = _categoryLevels.GetValueOrDefault(
            category,
            _categoryLevels.GetValueOrDefault("Default", LogLevel.Information));

        return _loggers.GetOrAdd(category,
            name => new DatabaseLogger(name, _writer, minLevel));
    }

    public void Dispose()
    {
        GC.SuppressFinalize(this);
    }
}
```

## The Logging Service

Create a background service that reads log entries from the channel and writes them to the SQLite database.

```csharp
using System.Diagnostics;
using System.Threading.Channels;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;

namespace DatabaseLogging;

public class DatabaseLoggerService(
    IServiceProvider provider,
    Channel<LogEntry> channel) : IDisposable
{
    private readonly ChannelReader<LogEntry> _reader = channel.Reader;
    private readonly ChannelWriter<LogEntry> _writer = channel.Writer;
    private readonly IServiceScope _scope = provider.CreateScope();
    private CancellationTokenSource? _cts;
    private Stopwatch _stopwatch = new();
    private Task? _task;

    // Starts the logging service to process log entries.
    public void Start(CancellationToken? token = null)
    {
        _stopwatch = Stopwatch.StartNew();
        _cts = token != null
            ? CancellationTokenSource.CreateLinkedTokenSource(token.Value)
            : new CancellationTokenSource();
        _task = Task.Run(ProcessLogQueueAsync, _cts.Token);
    }

    // Stops the logging service and waits for completion.
    public async Task StopAsync()
    {
        if (_cts is null || _task is null)
            throw new InvalidOperationException("Logging service not started.");

        _writer.Complete();
        await _task;
    }

    // Processes log entries from the channel and writes them to the database.
    private async Task ProcessLogQueueAsync()
    {
        if (_cts is null)
            throw new InvalidOperationException("Logging service not started.");

        var context = _scope.ServiceProvider.GetRequiredService<LoggingDbContext>();
        var buffer = new List<LogEntry>(50);
        var entries = 0;

        await foreach (var entry in _reader.ReadAllAsync(_cts.Token))
        {
            buffer.Add(entry);

            if (buffer.Count >= 50)
            {
                await context.Logs.AddRangeAsync(buffer, _cts.Token);
                await context.SaveChangesAsync(_cts.Token);
                entries += buffer.Count;
                buffer.Clear();
            }
        }

        if (buffer.Count > 0)
        {
            await context.Logs.AddRangeAsync(buffer, _cts.Token);
            await context.SaveChangesAsync(_cts.Token);
            entries += buffer.Count;
            buffer.Clear();
        }

        _stopwatch.Stop();

        await context.Logs.AddAsync(new LogEntry
        {
            EventId = new EventId(0, "LoggingSummary"),
            Level = LogLevel.Information,
            Category = "DatabaseLoggerService",
            Message = $"Logging session completed. Total entries: {entries}. Duration: {_stopwatch.ElapsedMilliseconds}ms.",
            Timestamp = DateTime.UtcNow
        }, _cts.Token);
        await context.SaveChangesAsync(_cts.Token);
    }

    // Disposes the logging service and its resources.
    public void Dispose()
    {
        if (_cts != null)
        {
            if (!_cts.IsCancellationRequested)
            {
                _cts.Cancel();
            }
            _cts.Dispose();
        }
        _scope.Dispose();
        GC.SuppressFinalize(this);
    }
}
```

## Putting It All Together

Finally, set up the dependency injection container to register the logging services and configure the logging provider.

### Program.cs

```csharp
using System.Threading.Channels;
using DatabaseLogging;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;

var configuration = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: false)
    .Build();

var provider = new ServiceCollection()
    .Configure<LoggingOptions>(configuration.GetSection("Logging").Bind)
    .AddDbContext<LoggingDbContext>(config =>
    {
        config.UseSqlite(configuration.GetConnectionString("LogDb"));
    })
    .AddLogging()
    .AddSingleton(Channel.CreateUnbounded<LogEntry>())
    .AddSingleton<ILoggerProvider, DatabaseLoggerProvider>()
    .AddSingleton<DatabaseLoggerService>()
    .BuildServiceProvider();

var cts = new CancellationTokenSource();
var logging = provider.GetRequiredService<DatabaseLoggerService>();

void OnCancelKeyPress(object? sender, ConsoleCancelEventArgs e)
{
    e.Cancel = true;
    cts.Cancel();
}

async Task InitializeDatabase(CancellationToken token = default)
{
    using var scope = provider.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<LoggingDbContext>();
    await context.Database.MigrateAsync(token);
}

Console.CancelKeyPress += OnCancelKeyPress;

try
{
    await InitializeDatabase(cts.Token);
    logging.Start(cts.Token);

    var logger = provider.GetRequiredService<ILogger<Program>>();

    logger.LogInformation("Example log message.");
}
catch (OperationCanceledException)
{
    // Expected on cancellation
}
finally
{
    await logging.StopAsync();
    if (!cts.IsCancellationRequested)
    {
        cts.Cancel();
    }
    cts.Dispose();
    logging.Dispose();
    Console.CancelKeyPress -= OnCancelKeyPress;
}
```

### appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning"
    }
  },
  "ConnectionStrings": {
    "LogDb": "Data Source=logs.db"
  }
}
```

### LoggingOptions

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;

namespace DatabaseLogging;

public class LoggingOptions
{
    [ConfigurationKeyName("LogLevel")]
    public Dictionary<string, LogLevel> LogLevel { get; set; } = [];
}

```

### Conclusion

Running the application will create the SQLite database (if it doesn't exist) and log messages to it. You can query the `logs` table to analyze the logged events.

```sql
SELECT * FROM logs;
```

| Id  | EventId                                                               | Level       | Category                                       | Message                                                                                                                                                                             | Exception | Timestamp                   |
| --- | --------------------------------------------------------------------- | ----------- | ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | --------------------------- |
| 1   | 20101:Microsoft.EntityFrameworkCore.Database.Command.CommandExecuted  | Information | Microsoft.EntityFrameworkCore.Database.Command | Executed DbCommand (7ms) [Parameters=[], CommandType='Text', CommandTimeout='30'] PRAGMA journal_mode = 'wal';                                                                      |           | 2025-10-25 22:46:41.7703115 |
| 2   | 20411:Microsoft.EntityFrameworkCore.Migrations.AcquiringMigrationLock | Information | Microsoft.EntityFrameworkCore.Migrations       | Acquiring an exclusive lock for migration application. See https://aka.ms/efcore-docs-migrations-lock for more information if this takes too long.                                  |           | 2025-10-25 22:46:41.7747824 |
| 3   | 20101:Microsoft.EntityFrameworkCore.Database.Command.CommandExecuted  | Information | Microsoft.EntityFrameworkCore.Database.Command | Executed DbCommand (2ms) [Parameters=[], CommandType='Text', CommandTimeout='30'] SELECT COUNT(*) FROM "sqlite_master" WHERE "name" = '__EFMigrationsLock' AND "type" = 'table';    |           | 2025-10-25 22:46:41.779204  |
| 4   | 20101:Microsoft.EntityFrameworkCore.Database.Command.CommandExecuted  | Information | Microsoft.EntityFrameworkCore.Database.Command | Executed DbCommand (2ms) [Parameters=[], CommandType='Text', CommandTimeout='30'] CREATE TABLE IF NOT EXISTS "__EFMigrationsLock" (...)                                             |           | 2025-10-25 22:46:41.7835752 |
| 5   | 20101:Microsoft.EntityFrameworkCore.Database.Command.CommandExecuted  | Information | Microsoft.EntityFrameworkCore.Database.Command | Executed DbCommand (0ms) [Parameters=[], CommandType='Text', CommandTimeout='30'] INSERT OR IGNORE INTO "__EFMigrationsLock"(...)                                                   |           | 2025-10-25 22:46:41.7868968 |
| 6   | 20101:Microsoft.EntityFrameworkCore.Database.Command.CommandExecuted  | Information | Microsoft.EntityFrameworkCore.Database.Command | Executed DbCommand (0ms) [Parameters=[], CommandType='Text', CommandTimeout='30'] CREATE TABLE IF NOT EXISTS "__EFMigrationsHistory" (...)                                          |           | 2025-10-25 22:46:41.8135863 |
| 7   | 20101:Microsoft.EntityFrameworkCore.Database.Command.CommandExecuted  | Information | Microsoft.EntityFrameworkCore.Database.Command | Executed DbCommand (0ms) [Parameters=[], CommandType='Text', CommandTimeout='30'] SELECT COUNT(*) FROM "sqlite_master" WHERE "name" = '__EFMigrationsHistory' AND "type" = 'table'; |           | 2025-10-25 22:46:41.8216622 |
| 8   | 20101:Microsoft.EntityFrameworkCore.Database.Command.CommandExecuted  | Information | Microsoft.EntityFrameworkCore.Database.Command | Executed DbCommand (0ms) [Parameters=[], CommandType='Text', CommandTimeout='30'] SELECT "MigrationId", "ProductVersion" FROM "__EFMigrationsHistory" ORDER BY "MigrationId";       |           | 2025-10-25 22:46:41.823541  |
| 9   | 20402:Microsoft.EntityFrameworkCore.Migrations.MigrationApplying      | Information | Microsoft.EntityFrameworkCore.Migrations       | Applying migration '20251025015921_InitializeLogs'.                                                                                                                                 |           | 2025-10-25 22:46:41.8279452 |
| 10  | 20101:Microsoft.EntityFrameworkCore.Database.Command.CommandExecuted  | Information | Microsoft.EntityFrameworkCore.Database.Command | Executed DbCommand (0ms) [Parameters=[], CommandType='Text', CommandTimeout='30'] CREATE TABLE "logs" (...)                                                                         |           | 2025-10-25 22:46:41.8320635 |
| 11  | 20101:Microsoft.EntityFrameworkCore.Database.Command.CommandExecuted  | Information | Microsoft.EntityFrameworkCore.Database.Command | Executed DbCommand (0ms) [Parameters=[], CommandType='Text', CommandTimeout='30'] INSERT INTO "__EFMigrationsHistory" (...)                                                         |           | 2025-10-25 22:46:41.8321002 |
| 12  | 20101:Microsoft.EntityFrameworkCore.Database.Command.CommandExecuted  | Information | Microsoft.EntityFrameworkCore.Database.Command | Executed DbCommand (1ms) [Parameters=[], CommandType='Text', CommandTimeout='30'] DELETE FROM "__EFMigrationsLock";                                                                 |           | 2025-10-25 22:46:41.8342862 |
| 13  | 0:                                                                    | Information | Program                                        | Example log message.                                                                                                                                                                |           | 2025-10-25 22:46:41.8364877 |
| 14  | 0:LoggingSummary                                                      | Information | DatabaseLoggerService                          | Logging session completed. Total entries: 13. Duration: 99ms.                                                                                                                       |           | 2025-10-25 22:46:41.9352682 |
