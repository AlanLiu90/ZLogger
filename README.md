ZLogger
===
[![GitHub Actions](https://github.com/Cysharp/ZLogger/workflows/Build-Debug/badge.svg)](https://github.com/Cysharp/ZLogger/actions) [![Releases](https://img.shields.io/github/release/Cysharp/ZLogger.svg)](https://github.com/Cysharp/ZLogger/releases)

**Z**ero Allocation Text/Structured **Logger** for .NET and Unity, with StringInterpolation and Source Generator, built on top of a `Microsoft.Extensions.Logging`.

The usual destinations for log output are `Console(Stream)`, `File(Stream)`, `Network(Stream)`, all in UTF8 format. However, since typical logging architectures are based on Strings (UTF16), this requires additional encoding costs. In ZLogger, we utilize the [String Interpolation Improvement of C# 10](https://devblogs.microsoft.com/dotnet/string-interpolation-in-c-10-and-net-6/) and by leveraging .NET 8's [IUtf8SpanFormattable](https://learn.microsoft.com/en-us/dotnet/api/system.iutf8spanformattable?view=net-8.0), we have managed to avoid the boxing of values and maintain high performance by consistently outputting directly in UTF8 from input to output.

ZLogger is built directly on top of `Microsoft.Extensions.Logging`. `Microsoft.Extensions.Logging` is an official log abstraction used in many frameworks, such as ASP.NET Core and Generic Host. However, since regular loggers have their own systems, a bridge is required to connect these systems, and this is where a lot of overhead can be observed. ZLogger eliminates the need for this bridge, thereby completely avoiding overhead.

![Alt text](docs/image.png)

This benchmark is for writing to a file, but the default settings of typical loggers are very slow. This is because they flush after every write. In the benchmark, to ensure fairness, careful attention was paid to set the options in each logger for maximum speed. ZLogger is designed to be the fastest by default, so there is no need to worry about any settings.

The slowness of this default setting is due to I/O, so it can be mitigated by using a faster drive. When taking benchmarks, please note that the results can vary greatly not only on your local (which is probably fast!) but also on drives attached to the cloud and in environments like Docker. One of the good points about the async-buffered setting is that it can reduce the impact of such I/O issues.

ZLogger focuses on the new syntax of C#, and fully adopts Interpolated Strings.

![Alt text](docs/image-1.png)

This allows for providing parameters to logs in the most convenient form. Also, by closely integrating with System.Text.Json's Utf8JsonWriter, it not only enables high-performance output of text logs but also makes it possible to efficiently output structured logs.

ZLogger also emphasizes console output, which is crucial in cloud-native applications. By default, it outputs with performance that can withstand destinations in cloud log management. Of course, it supports both text logs and structured logs.

ZLogger delivers its best performance with .NET 8 and above, but it is designed to maintain consistent performance with .NET Standard 2.0 and .NET 6 through a fallback to its own IUtf8SpanFormattable.

As for standard logger features, it supports loading LogLevel from json, filtering by category, and scopes, as found in Microsoft.Extensions.Logging. In terms of output destinations, it is equipped with sufficient capabilities for `Console`, `File`, `RollingFile`, `InMemory`, `Stream`, and an `AsyncBatchingProcessor` for sending logs over HTTP and similar protocols.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of Contents

- [Getting Started](#getting-started)
- [Formatter Configurations](#formatter-configurations)
  - [PlainText](#plaintext)
  - [JSON](#json)
  - [MessagePack](#messagepack)
  - [Custom Formatter](#custom-formatter)
- [LogInfo](#loginfo)
- [KeyNameMutator](#keynamemutator)
- [ZLoggerBuilder](#zloggerbuilder)
  - [Console](#console)
  - [File](#file)
  - [RollingFile](#rollingfile)
  - [Stream](#stream)
  - [In-Memory](#in-memory)
  - [Custom LogProcessor](#custom-logprocessor)
- [ZLoggerOptions](#zloggeroptions)
- [Custom Format](#custom-format)
- [ZLoggerMessage Source Generator](#zloggermessage-source-generator)
- [Microsoft.CodeAnalysis.BannedApiAnalyzers](#microsoftcodeanalysisbannedapianalyzers)
- [Global LoggerFactory](#global-loggerfactory)
- [Unity](#unity)
  - [Installation](#installation)
  - [Basic usage](#basic-usage)
- [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

Getting Started
---
This library is distributed via NuGet, supporting `.NET Standard 2.0`, `.NET Standard 2.1`, `.NET 6(.NET 7)` and `.NET 8` or above.

> PM> Install-Package [ZLogger](https://www.nuget.org/packages/ZLogger)

In the simplest case, you generate a logger by adding ZLogger's Provider to Microsoft.Extensions.Logging's [LoggerFactory](https://learn.microsoft.com/en-us/dotnet/core/extensions/logging), and then use ZLogger's own ZLog method.

For Unity, the requirements and installation process are completely different. See the [Unity](#unity) section for details.


```csharp
using Microsoft.Extensions.Logging;
using ZLogger;

using var factory = LoggerFactory.Create(logging =>
{
    logging.SetMinimumLevel(LogLevel.Trace);

    // Add ZLogger provider to ILoggingBuilder
    logging.AddZLoggerConsole();
    
    // Output Structured Logging, setup options
    // logging.AddZLoggerConsole(options => options.UseJsonFormatter());
});

var logger = factory.CreateLogger("Program");

var name = "John";
var age = 33;

// Use **Z**Log method and string interpolation to log message
logger.ZLogInformation($"Hello my name is {name}, {age} years old.");
```

Normally, you don't create LoggerFactory yourself. Instead, you set up a Generic Host and receive ILogger through dependency injection (DI).

You can setup logger by [.NET Generic Host](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host?view=aspnetcore-8.0)(for ASP.NET Core) and if you want to use this in ConsoleApplication, we provides [ConsoleAppFramework](https://github.com/Cysharp/ConsoleAppFramework) to use hosting abstraction.

Here is the showcase of providers.

```csharp
using ZLogger;

var builder = Host.CreateApplicationBuilder();

builder.Logging
    // optional(MS.E.Logging):clear default providers(recommend to remove all)
    .ClearProviders()

    // optional(MS.E.Logging):setup minimum log level
    .SetMinimumLevel(LogLevel.Trace)
    
    // Add to output to console
    .AddZLoggerConsole();

    // Add to output to the file
    .AddZLoggerFile("/path/to/file.log")
    
    // Add to output the file that rotates at constant intervals.
    .AddZLoggerRollingFile(options =>
    {
        // File name determined by parameters to be rotated
        options.FilePathSelector = (timestamp, sequenceNumber) => $"logs/{timestamp.ToLocalTime():yyyy-MM-dd}_{sequenceNumber:000}.log";
        
        // The period of time for which you want to rotate files at time intervals.
        options.RollingInterval = RollingInterval.Day;
        
        // Limit of size if you want to rotate by file size. (KB)
        options.RollingSizeKB = 1024;        
    })    
    
    // Add to output of simple rendered strings into memory. You can subscribe to this and use it.
    .AddZLoggerInMemory(processor =>
    {
        processor.MessageReceived += renderedLogString => 
        {
            System.Console.WriteLine(renderedLogString);    
        };
    })
    
    // Add output to any steram (`System.IO.Stream`)
    .AddZLoggerStream(stream);

    // Add custom output
    .AddZLoggerLogProcessor(new YourCustomLogExporter());
    
    // Format as json
    .AddZLoggerConsole(options =>
    {
        options.UseJsonFormatter();
    })
    
    // Format as json and configure output
    .AddZLoggerConsole(options =>
    {
        options.UseJsonFormatter(formatter =>
        {
            formatter.IncludeProperties = IncludeProperties.ParameterKeyValues;
        });
    })

    // Further common settings
    .AddZLoggerConsole(options =>
    {
        // Enable LoggerExtensions.BeginScope
        options.IncludeScopes = true;
        
        // Set TimeProvider
        options.TimeProvider = yourTimeProvider
    });
```

```cs
using Microsoft.Extensions.Logging;
using ZLogger;

public class MyClass
{
    // get ILogger<T> from DI.
    readonly ILogger<MyClass> logger;
    
    public MyClass(ILogger<MyClass> logger)
    {
        this.logger = logger;
    }
    
    // name = "Bill", city = "Kumamoto", age = 21
    public void Foo(string name, string city, int age)
    {
        // plain-text:
        // Hello, Bill lives in Kumamoto 21 years old.
        // json:
        // {"Timestamp":"2023-11-30T17:28:35.869211+09:00","LogLevel":"Information","Category":"MyClass","Message":"Hello, Bill lives in Kumamoto 21 years old.","name":"Bill","city":"Kumamoto","age":21}
        // json(IncludeProperties.ParameterKeyValues):
        // {"name":"Bill","city":"Kumamoto","age":21}
        logger.ZLogInformation($"Hello, {name} lives in {city} {age} years old.");
    
        // Explicit property name, you can use custom format string start with '@'
        logger.ZLogInformation($"Hello, {name:@user-name} id:{100:@id} {age} years old.");
    
        // Dump variables as JSON, you can use custom format string `json`
        var user = new User(1, "Alice");

        // user: {"Id":1,"Name":"Bob"}
        logger.ZLogInformation($"user: {user:json}");
    }
}
```

All standard `.Log` methods are processed as strings by ZLogger's Provider. However, by using our unique `.ZLog*` methods, you can process them at high performance while remaining in UTF8. Additionally, these methods support both text logs and structured logs using String Interpolation syntax.

All logging methods are completely similar as [Microsoft.Extensions.Logging.LoggerExtensions](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.loggerextensions), but it has **Z** prefix overload.

The ZLog* method uses [InterpolatedStringHandler](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/tutorials/interpolated-string-handler) in .NET and prepare the template at compile time.

Formatter Configurations
----

// TODO: still being written, please wait a little longer.


### PlainText

```cs
builder.Logging.AddZLoggerConsole(options =>
{
    // Text format
    // e.g) "2023-12-01 16:41:55.775|Information|This is log message. (MyNamespace.MyApp)
    options.UsePlainTextFormatter(formatter => 
    {
        formatter.SetPrefixFormatter($"{0}|{1}|", (in MessageTemplate template, in LogInfo info) => template.Format(info.Timestamp, info.LogLevel));
        formatter.SetSuffixFormatter($" ({0})", (in MessageTemplate template, in LogInfo info) => template.Format(info.Category));
        formatter.SetExceptionFormatter((writer, ex) => Utf8String.Format(writer, $"{ex.Message}"));
    });
        
    // Using various variable formats.
    // e.g) "2023-12-01T16:47:15+09:00|INF|This is log message"
    formatter.SetPrefixFormatter($"{0:yyyy-MM-dd'T'HH:mm:sszzz}|{1:short}|", (in MessageTemplate template, in LogInfo info) =>
    {
        var escapeSequence = "";
        // if (info.LogLevel >= LogLevel.Error)
        // {
        //     escapeSequence = "\u001b[31m";
        // }
        // else if (!info.Category.Name.Contains("MyApp"))
        // {
        //     escapeSequence = "\u001b[38;5;08m";
        // }
    
        template.Format(info.Timestamp, info.LogLevel);
    });
        
    // Console coloring example
    options.UsePlainTextFormatter(formatter =>
    {
        // \u001b[31m => Red(ANSI Escape Code)
        // \u001b[0m => Reset
        // \u001b[38;5;***m => 256 Colors(08 is Gray)
        formatter.SetPrefixFormatter($"{0}{1}|{2:short}|", (in MessageTemplate template, in LogInfo info) =>
        {
            var escapeSequence = "";
            if (info.LogLevel >= LogLevel.Error)
            {
                escapeSequence = "\u001b[31m";
            }
            else if (!info.Category.Name.Contains("MyApp"))
            {
                escapeSequence = "\u001b[38;5;08m";
            }
        
            template.Format(escapeSequence, info.Timestamp, info.LogLevel);
        });

        formatter.SetSuffixFormatter($"{0}", (in MessageTemplate template, in LogInfo info) =>
        {
            if (info.LogLevel == LogLevel.Error || !info.Category.Name.Contains("MyApp"))
            {
                template.Format("\u001b[0m");
            }
        });
    });
});
```

Formatting can be set using the String Interpolation Template, and lambda expression as shown above.

Note: For format strings available for various variables:
-`LogLevel` can be specially specified as `short`. This reduces the length of string to a fixed number of characters, such as `INFO`.
- For other types, ZLogger uses [Cysharp/Utf8StringInterpolation](https://github.com/Cysharp/Utf8StringInterpolation) internally. Please see this.

`public delegate void MessageTemplateFormatter(in MessageTemplate template, in LogInfo info);`

| Name                                                                                             | Description                                                          |
|:-------------------------------------------------------------------------------------------------|:---------------------------------------------------------------------|
| `SetPrefixFormatter(MessageTemplateHandler format, MessageTemplateFormatter formatter)`  | Set the text to be given before the message body. (Default is empty) |
| `SetSuffixFormatter(MessageTemplateHandler format, MessageTemplateFormatter formatter)`  | Set the text to be given after the message body. (Default is empty)  |
| `SetExceptionFormatter(Action<IBufferWriter<byte>, Exception> formatter)`                        |                                                                      |



### JSON

`public delegate void JsonLogInfoFormatter(Utf8JsonWriter jsonWriter, in LogInfo info);`

| Name                                                                | Description                                                                       |
|:--------------------------------------------------------------------|:----------------------------------------------------------------------------------|
| `JsonPropertyNames JsonPropertyNames`                               | Specify the name of each key in the output JSON                                   |
| `IncludeProperties IncludeProperties`                               | Flags that can specify properties to be output. (default: `Timestamp              | LogLevel | CategoryName | Message | Exception | ScopeKeyValues | ParameterKeyValues`) |
| `JsonSerializerOptions JsonSerializerOptions`                       | The options of `System.Text.Json`                                                 |
| `JsonLogInfoFormatter? AdditionalFormatter`                         | Action when rendering additional properties based on `LogInfo`.                   |
| `JsonEncodedText? PropertyKeyValuesObjectName`                      | If set, the key/value properties is nested under the specified key name.          |
| `IKeyNameMutator? KeyNameMutator`                                   | You can set the naming convention if you want to automatically convert key names. |
| `bool UseUtcTimestamp`                                              | If true, timestamp is output in utc. (default: false)                             |


Sample of Json Formatting customize

```csharp
using System.Text.Json;
using ZLogger;
using ZLogger.Formatters;

namespace ConsoleApp;

using static IncludeProperties;
using static JsonEncodedText; // JsonEncodedText.Encode

public static class CloudLoggingExtensions
{
    // Cloud Logging Json Field
    // https://cloud.google.com/logging/docs/structured-logging?hl=en
    public static ZLoggerOptions UseCloudLoggingJsonFormat(this ZLoggerOptions options)
    {
        return options.UseJsonFormatter(formatter =>
        {
            // Category and ScopeValues is manually write in AdditionalFormatter at labels so remove from include properties.
            formatter.IncludeProperties = Timestamp | LogLevel | Message | ParameterKeyValues;

            formatter.JsonPropertyNames = JsonPropertyNames.Default with
            {
                LogLevel = Encode("severity"),
                LogLevelNone = Encode("DEFAULT"),
                LogLevelTrace = Encode("DEBUG"),
                LogLevelDebug = Encode("DEBUG"),
                LogLevelInformation = Encode("INFO"),
                LogLevelWarning = Encode("WARNING"),
                LogLevelError = Encode("ERROR"),
                LogLevelCritical = Encode("CRITICAL"),

                Message = Encode("message"),
                Timestamp = Encode("timestamp"),
            };

            formatter.PropertyKeyValuesObjectName = Encode("jsonPayload");

            // cache JsonENcodedText outside of AdditionalFormatter
            var labels = Encode("logging.googleapis.com/labels");
            var category = Encode("category");
            var eventId = Encode("eventId");
            var userId = Encode("userId");

            formatter.AdditionalFormatter = (Utf8JsonWriter writer, in LogInfo) =>
            {
                writer.WriteStartObject(labels);
                writer.WriteString(category, logInfo.Category.JsonEncoded);
                writer.WriteString(eventId, logInfo.EventId.Name);

                if (logInfo.ScopeState != null && !logInfo.ScopeState.IsEmpty)
                {
                    foreach (var item in logInfo.ScopeState.Properties)
                    {
                        if (item.Key == "userId")
                        {
                            writer.WriteString(userId, item.Value!.ToString());
                            break;
                        }
                    }
                }
                writer.WriteEndObject();
            };
        });
    }
}

```

### MessagePack


Formats using messagepack are supported in an additional package.

[MessagePack-CSharp](https://github.com/MessagePack-CSharp/MessagePack-CSharp)

> PM> Install-Package [ZLogger.MessagePack](https://www.nuget.org/packages/ZLogger.MessagePack)

| Name                                                               | Description                                                        |
|:-------------------------------------------------------------------|:-------------------------------------------------------------------|
| `MessagePackSerializerOptions MessagePackSerializerOptions`        | The options of `MessagePack-CSharp`.                               |
| `IncludeProperties IncludeProperties`                              | Flags that can specify properties to be output. (default: `Timestamp| LogLevel | CategoryName | Message | Exception | ScopeKeyValues | ParameterKeyValues`) |
| `IKeyNameMutator? KeyNameMutator`                                   | You can set the naming convention if you want to automatically convert key names. |

### Custom Formatter 

todo

LogInfo
---

| Name                        | Description                                                                                              |
|:----------------------------|:---------------------------------------------------------------------------------------------------------|
| `LogCategory Category`      | The category name set for each logger. And holds JsonEncodedText and utf8 byte sequence representations. |
| `Timestamp Timestamp`       | Timestamp                                                                                                |
| `LogLevel LogLevel`         | LogLevel  of `Microsoft.Extensions.Logging`                                                              |
| `EventId EventId`           | EventId of `Microsoft.Extensions.Logging`                                                                |
| `Exception? Exception`      | Exception given as argument when logging.                                                                |
| `LogScopeState? ScopeState` | Additional properties set by `ILogger.BeginScope(...)` (if ZLoggerOptions.IncludeScopes = true)          |
| `object? Context`    | Additional context | 
| `string? MemberName` | Caller MemberName         |
| `string? FilePath` | Caller FilePath         |
| `int LineNumber` | Caller LineNumber         |


KeyNameMutator
---


| Name                                  | Description                                                                                               |
|:--------------------------------------|:----------------------------------------------------------------------------------------------------------|
| `LastMemberName`                      | Returns the last member name of the source.                                                               |
| `LowerFirstCharacter`                 | The first character converted to lowercase.                                                               |
| `UpperFirstCharacter`                 | The first character converted to uppercase.                                                               |
| `LastMemberNameLowerFirstCharacter`   | Returns the last member name of the source with the first character converted to lowercase.               |
| `LastMemberNameUpperFirstCharacter`   | Returns the last member name of the source with the first character converted to uppercase.               |                              







ZLoggerBuilder
----

ZLogger has the following providers.

| Type                                   | Alias               | Builder Extension      |
|----------------------------------------|---------------------|------------------------|
| ZLoggerConsoleLoggerProvider           | ZLoggerConsole      | AddZLoggerConsole      |
| ZLoggerFileLoggerProvider              | ZLoggerFile         | AddZLoggerFile         |
| ZLoggerRollingFileLoggerProvider       | ZLoggerRollingFile  | AddZLoggerRollingFile  |
| ZLoggerStreamLoggerProvider            | ZLoggerStream       | AddZLoggerStream       |
| ZLoggerLogProcessorLoggerProvider      | ZLoggerLogProcessor | AddZLoggerLogProcessor |
| ZLoggerInMemoryProcessorLoggerProvider | ZLoggerInMemory     | AddZLoggerInMemory     |


If you are using `Microsoft.Extensions.Configuration`, you can set the log level through configuration.
In this case, alias of Provider can be used.  for example:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    },
    "ZLoggerConsoleLoggerProvider": {
      "LogLevel": {
        "Default": "Debug"
      }
    }
  }
}
```

All Providers can take an Action that sets ZLoggerOptions as the last argument. As follows.

```cs
builder.Logging
    .ClearProviders()
    // Configure options
    .AddZLoggerConsole(options => 
    {
        options.LogToStandardErrorThreshold = LogLevel.Error;
    });
    
    // Configure options with service provider
    .AddZLoggerConsole((options, services) => 
    {
        options.TimeProvider = services.GetService<YourCustomTimeProvider>();
    });
```


### Console


If you are using `ZLoggerConsoleLoggerProvider`, the following additional options are available:

| Name                                    | Description                                                                                                                               |
|:----------------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------|
| `bool OutputEncodingToUtf8`             | Set `Console.OutputEncoding = new UTF8Encoding(false)` when the provider is created.  (default: true)                                     |
| `bool ConfigureEnableAnsiEscapeCode`    | If set true, then configure console option on execution and enable virtual terminal processing(enable ANSI escape code). (default: false) |
| `LogLevel LogToStandardErrorThreshold`  | If set, logs at a higher level than the value will be output to standard error. (default: LogLevel.None)                                  |


### File


If you are using `ZLoggerRollingFileLoggerProvider`, the following additional options are available:

| Name                                                                              | Description                                                                                                        |
|:----------------------------------------------------------------------------------|:-------------------------------------------------------------------------------------------------------------------|
| `bool fileShared`                                                                  | If set true, enables exclusive control of writing to the same file from multiple processes.(default: false) |


### RollingFile


If you are using `ZLoggerRollingFileLoggerProvider`, the following additional options are available:

| Name                                                                              | Description                                                                                                        |
|:----------------------------------------------------------------------------------|:-------------------------------------------------------------------------------------------------------------------|
| `Func<DateTimeOffset, int, string> fileNameSelector`                              | The Func to consturct the file path. `DateTimeOffset` is date of file open time(UTC), `int` is number sequence.        |
| `RollingInterval rollInterval`                                                    | Interval to automatically rotate files.                                                                            |
| `int rollSizeKB`                                                                  | Limit size of single file.  If the file size is exceeded, a new file is created with the sequence number moved up. |
| `bool fileShared`                                                                  | If set true, enables exclusive control of writing to the same file from multiple processes.(default: false) |


### Stream

```cs
builder.Logging
    .ClearProviders()
    .AddZLogger(zlogger => 
    {
        // Default
        zlogger.AddStream(stream);
        
        // Configure stream dynamically
        zlogger.AddStream((options, services) =>
        {
            // ...
            return yourCustomStream;
        });
    });

```

### In-Memory



| Name                                                                                                            | Description |
|:----------------------------------------------------------------------------------------------------------------|:------------|
| `string processorKey`                                                                                           |  If specified, `InMemoryObservableLogProcessor` is registered in the DI container as a keyed service and can be retrieved by name.           |
| `Action<InMemoryObservableLogProcessor> configureProcessor`                                                     |  Custom actions can be added that use processors instead of DI containers.           |



### Custom LogProcessor

todo


```cs

public class TcpLogProcessor : IAsyncLogProcessor
{
    TcpClient tcpClient;
    AsyncStreamLineMessageWriter writer;

    public TcpLogProcessor(ZLoggerOptions options)
    {
        tcpClient = new TcpClient("127.0.0.1", 1111);
        writer = new AsyncStreamLineMessageWriter(tcpClient.GetStream(), options);
    }

    public void Post(IZLoggerEntry log)
    {
        writer.Post(log);
    }

    public async ValueTask DisposeAsync()
    {
        await writer.DisposeAsync();
        tcpClient.Dispose();
    }
}
```

```cs
public class BatchingHttpLogProcessor : BatchingAsyncLogProcessor
{
    HttpClient httpClient;
    ArrayBufferWriter<byte> bufferWriter;
    IZLoggerFormatter formatter;

    public BatchingHttpLogProcessor(int batchSize, ZLoggerOptions options)
        : base(batchSize, options)
    {
        httpClient = new HttpClient();
        bufferWriter = new ArrayBufferWriter<byte>();
        formatter = options.CreateFormatter();
    }

    protected override async ValueTask ProcessAsync(IReadOnlyList<INonReturnableZLoggerEntry> list)
    {
        foreach (var item in list)
        {
            item.FormatUtf8(bufferWriter, formatter);
        }
        
        var byteArrayContent = new ByteArrayContent(bufferWriter.WrittenSpan.ToArray());
        await httpClient.PostAsync("http://foo", byteArrayContent).ConfigureAwait(false);

        bufferWriter.Clear();
    }

    protected override ValueTask DisposeAsyncCore()
    {
        httpClient.Dispose();
        return default;
    }
}
```


ZLoggerOptions
---


| Name                                                                         | Description                                                                                                                                                                                                                    |
|:-----------------------------------------------------------------------------|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `bool IncludeScopes { get; set; }`                                           | Enable `ILogger.BeginScope`, default is `false`.                                                                                                                                                                               |
| `TimeProvider? TimeProvider { get; set; }`                                   | Gets or sets the time provider for the logger. The Timestamp of LogInfo is generated by TimeProvider's GetUtcNow() and LocalTimeZone when TimeProvider is set. The default value is null, which means use the system standard. |
| `Action<Exception>? InternalErrorLogger { get; set; }`                       | `InternalErrorLogger` is a delegate that is called when an exception occurs in the log writing process (such as a serialization error). The default value is `null`, which means errors are ignored.                           |
| `CreateFormatter()`                                                          | Create an formatter to use in ZLoggerProvider.                                                                                                                                                                                 |
| `UseFormatter(Func<IZLoggerFormatter> formatterFactory)`                     | Set the formatter that defines the output format of the log.                                                                                                                                                                   |
| `UsePlainTextFormatter(Action<PlainTextZLoggerFormatter>? configure = null)` | Use the built-in plain text formatter.                                                                                                                                                                                         |
| `UseJsonFormatter(Action<SystemTextJsonZLoggerFormatter>? configure = null)` | Use the built-in json formatter. (implementation of `System.Text.Json`)                                                                                                                                                        |

TODO:...
default formatter is PlaintTextFormatter.


Custom Format
---
TODO:///

`@`, `json`


ZLoggerMessage Source Generator
---
TODO:/.///

```csharp
public static partial class MyLogger
{
    [ZLoggerMessage(LogLevel.Information, "Bar: {x} {y}")]
    public static partial void Bar(this ILogger<Foo> logger, int x, int y);
}
```

Microsoft.CodeAnalysis.BannedApiAnalyzers
---
[Microsoft.CodeAnalysis.BannedApiAnalyzers](https://github.com/dotnet/roslyn-analyzers/blob/master/src/Microsoft.CodeAnalysis.BannedApiAnalyzers/BannedApiAnalyzers.Help.md) is an interesting analyzer, you can prohibit the normal Log method and induce the user to call ZLogger's ZLog method.

![image](https://user-images.githubusercontent.com/46207/78545188-56ea8a80-7836-11ea-81f2-6cbf7119f027.png)

All you have to do is prepare the following configuration.

```
T:Microsoft.Extensions.Logging.LoggerExtensions;Don't use this, use ZLog*** instead.
T:System.Console;Don't use this, use logger instead.
```

Global LoggerFactory
---
Like the traditional log manager, how to get and store logger per type without DI(such as `static readonly ILogger logger = LogManager.GetLogger()`). You can get `ILoggerFactory` from `IHost` before Run and set to the global static loggerfactory store.

```csharp
var host = Host.CreateDefaultBuilder()
    .ConfigureLogging(logging =>
    {
        logging.ClearProviders();
        logging.AddZLoggerConsole();
    })
    .UseConsoleAppFramework<Program>(args) // use framework, example of ConsoleAppFramework
    // .ConfigureWebHostDefaults(x => x.UseStartup<Startup>()) // example of ASP.NET Core
    .Build(); // use Build instead of Run directly

// get configured loggerfactory.
var loggerFactory = host.Services.GetRequiredService<ILoggerFactory>();

LogManager.SetLoggerFactory(loggerFactory, "Global");

// Run after set global logger.
await host.RunAsync();

// -----

// Own static logger manager
public static class LogManager
{
    static ILogger globalLogger;
    static ILoggerFactory loggerFactory;

    public static void SetLoggerFactory(ILoggerFactory loggerFactory, string categoryName)
    {
        LogManager.loggerFactory = loggerFactory;
        LogManager.globalLogger = loggerFactory.CreateLogger(categoryName);
    }

    public static ILogger Logger => globalLogger;

    // standard LoggerFactory caches logger per category so no need to cache in this manager
    public static ILogger<T> GetLogger<T>() where T : class => loggerFactory.CreateLogger<T>();
    public static ILogger GetLogger(string categoryName) => loggerFactory.CreateLogger(categoryName);
}
```

You can use this logger manager like following.

```csharp
public class Foo
{
    static readonly ILogger<Foo> logger = LogManager.GetLogger<Foo>();

    public void Foo(int x)
    {
        logger.ZLogDebug($"do do do: {x}");
    }
}
```

Unity
---

### Installation

ZLogger uses some of the compile time features of C# 10, and ZLogger.Generator uses some of the features of C# 11.

To use them in Unity, needs to check the Unity version and set up the compiler.

- Unity 2022.2 or newer
  - Standard ZLogger features are available.
  - Unity internally embeds the .NET SDK 6. So C# 10 is available via compiler arguments.
- Unity 2022.3.12f1 or newer
  - ZLogger source generator available.
  - Unity internaly update .NET SDK 6. So C# 11 features are in preview.

Prerequirements:
- Install [NuGetForUnity](https://github.com/GlitchEnzo/NuGetForUnity)
  - Required to install the dlls of ZLogger and its dependencies.
- Install [CsprojModifier](https://github.com/Cysharp/CsprojModifier) 
  - Required to develop in the IDE with a new language version.
- Install `ZLogger.Unity` package via git url.
  - Add `https://github.com/Cysharp/ZLogger.git?path=src/ZLogger.Unity/Assets/ZLogger.Unity` to Package Manager
  
Installation steps:

1. Setup the C# compiler for unity. 
    - Add a text file named `csc.rsp` with the following contents under your Assets/.
        - ```
          -langVersion:10 -nullable
          ```
    - Note:
        - If you are using assembly definition, put it in the same folder as the asmdef that references ZLogger.
        - If you are using Unity 2022.3.12f1 or newer, you can use `langVersion:preview` allows parts of C# 11 features.

2. Setup the C# compiler for your IDE. 
    - Add a text file named LangVersion.props with the following contents
        - ```xml
          <Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
            <PropertyGroup>
              <LangVersion>10</LangVersion>
              <Nullable>enable</Nullable>
            </PropertyGroup>
          </Project>
          ``` 
    - Open Project Settings and [C# Project Modifier] section under the [Editor].
    - Add the .props file you just created, to the list of [Additional project imports].
    - Note:
        - If you are using assembly definition, add your additional csproj in the list of [The project to be addef for import].
        - If you want to use `ZLoggerMessage` Source Generator, require Unity 2022.3.12f1 and change to `<LangVersion>11</LangVersion>`
3. Install ZLogger nuget package. 
    - Open [Nuget] -> [Manage Nuget Packages] in the menu bar.
    - Search `ZLogger`, and press [Install].



### Basic usage

The basic functions of ZLogger are also available in Unity as follows. Use LoggerFactory directly to create loggers.

```cs
var loggerFactory = LoggerFactory.Create(logging =>
{
    logging.SetMinimumLevel(LogLevel.Trace);
    logging.AddZLoggerUnityDebug(); // log to UnityDebug
});

var logger = loggerFactory.CreateLogger<YourClass>();

var name = "foo";
logger.ZLogInformation($"Hello, {name}!");
```

Also supports StructuredLogging(JSON), and FileProvider.

```cs
var loggerFactory = LoggerFactory.Create(logging =>
{
    logging.AddZLoggerFile("/path/to/logfile", options =>
    {
        options.UseJsonFormatter();
    });
});
```

Unity 2022.3.12f1 and enables `-langVersion:preview` supports Source Generator.

```csharp
public static partial class LogExtensions
{
    [ZLoggerMessage(LogLevel.Debug, "Hello, {name}")]
    public static partial void Hello(this ILogger<NewBehaviourScript> logger, string name);
}
```

License
---
This library is licensed under the MIT License.
