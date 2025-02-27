---
description: How to extend squidex and the different extension points for customization's.
---

# Extensions

## Introduction

This document describes how to write extensions for Squidex. We assume that you know the Basic principles of C# and ASP.NET Core. Otherwise it might be very hard to write custom extensions.

We use interfaces for all components, which can then be replaced with custom implementations. The default dependency injection system is used to register services. (see [https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection)).

Furthermore you can extend Squidex with custom HTTP endpoints. Due to the static nature of the Management UI which is build with Typescript and Angular and has a compilation and bundling process it you cannot extend the UI, except in a few cases where it has been made possible.

Read the following article to understand how to write custom editors for the Management UI:

{% content-ref url="../../../02-documentation/developer-guides/editors.md" %}
[editors.md](../../../02-documentation/developer-guides/editors.md)
{% endcontent-ref %}

## How to write a custom Plugin

So far (October 2019) Squidex has a limited plugin architecture, that can be used for most cases. In this section I will give you an introduction how to write a such a plugin and how to register it in Squidex.

In this tutorial we use `Squidex.Extensions` project as an example, which also contains a sample plugin.

### 1. Clone the source code

Just checkout the source code from Github:

```
git@github.com:Squidex/squidex.git
```

You only need the source code for Development. There are other approaches how to deploy the newest version of Squidex with your plugin.

### 2. Create a .NET Core class library

First you have to create a new .NET class library to the backend solution:

![Squidex.Extensions Plugin](<../../../.gitbook/assets/image (19).png>)

Ensure that you target `netcoreapp3.0`.

Also add references to the following other projects in the solution:

1. `Squidex.Domain.Apps.Entities`
2. `Squidex.Web`

This will implicitly create references to all other projects that might be needed.

{% hint style="info" %}
The references projects contain interfaces as well as implementations, which is a problem, because it is hard to figure out which classes should be used in custom plugins. Implementations are subject to change and should not be used directly. Therefore we plan to restructure the content and to create implicit contract projects, which do not contain implementation details.
{% endhint %}

### 3. Create a new Plugin class.

A plugin class is used to register all implementations to Squidex. In this example we create a custom plugin that registers an asset store that keeps all files in Memory. We register it only when it is activated via the configuration system, so either from `appSettings.json`, command line arguments or environment variables.

```csharp
namespace Squidex.Extensions.Samples.AssetStore
{
    public sealed class MemoryAssetStorePlugin : IPlugin
    {
        public void ConfigureServices(IServiceCollection services, IConfiguration config)
        {
            var storeType = config.GetValue<string>("assetStore:type");

            if (string.Equals(storeType, "Memory", StringComparison.OrdinalIgnoreCase))
            {
                services.AddSingletonAs<MemoryAssetStore>()
                    .As<IAssetStore>();
            }
        }
    }
}
```

&#x20;You can also add multiple implementations to your class library in case you have multiple extensions in one project and want to group them together.

### 3. Reference your plugin

Add a reference to your plugin to the `Squidex` project.

![Add reference to your plugin](<../../../.gitbook/assets/image (2) (1).png>)

Furthermore you need to add the path to your plugin to the configuration, for example to the `appSettings`file. Because we reference the plugin, it will be automatically copied to the output folder when the build or packaging is executed.

![Reference your plugin in the configuration file](<../../../.gitbook/assets/image (28) (1).png>)

There is also a project on Github that demonstrates how to create a plugin for SendGrid with an older version of Squidex: [https://github.com/squidexcontrib/sendgrid](https://github.com/squidexcontrib/sendgrid)

## Extension Points

There are a few extension points that can be used for custom functionality.

### Rule Actions

Rule actions are used to integrate external systems to Squidex. Therefore we have written a guide to show you how to write your first rule action.

{% content-ref url="how-to-write-custom-rule-actions.md" %}
[how-to-write-custom-rule-actions.md](how-to-write-custom-rule-actions.md)
{% endcontent-ref %}

### Controllers

You can write your own controllers, but you have to derive from `ApiController` class. Your controllers are available under the `/api` path, because all other requests will be forwarded to the Management UI. The `ApiController`enforces this rule and ensures the correct routing.

```csharp
public sealed class PluginController : ApiController
{
    public PluginController(ICommandBus commandBus)
        : base(commandBus)
    {
    }

    [Route("/plugins/sample")]
    public IActionResult Test()
    {
        return Ok(new { text = "I am Plugin" });
    }
}
```

### Infrastructure Extensions

#### IAssetStore

The `Squidex.Infrastructure.Assets.IAssetStore` interface is used to encapsulate storage solutions for assets. Currently there are the following implementations:

* `AzureBlobAssetStore`: Stores the assets in azure blob storage.
  * Read more: [https://azure.microsoft.com/en-us/services/storage/blobs/](https://azure.microsoft.com/en-us/services/storage/blobs/)
* `GoogleCloudAssetStore`: Stores the assets in Google cloud.
  * Read more: [https://cloud.google.com/storage/](https://cloud.google.com/storage/)``
* `MongoGridFsAssetStore`: Stores the assets in MongoDB using GridFS.
  * Read more: [https://docs.mongodb.com/manual/core/gridfs/](https://docs.mongodb.com/manual/core/gridfs/)
* `FolderAssetStore`: Stores the assets in the file system.

Recommended implementations:

* Amazon S3

#### IEventStore

The `Squidex.Infrastructure.CQRS.Events.IEventStore` is our abstraction for different event store implementations. You can append to events, query them or subscribe to events. Dependending on your implementation you might want to use the pub-sub system for subscriptions. The notification mechanism is provided by the `Squidex.Infrastructure.CQRS.Events.IEventNotifier` interface. Currently there are the following implementations:

* `Squidex.Infrastructure.CQRS.Events.MongoEventStore`: Implementation for MongoDb.&#x20;
  * Read more: [https://docs.mongodb.com/ecosystem/drivers/csharp/](https://docs.mongodb.com/ecosystem/drivers/csharp/)
* `Squidex.Infrastructure.CQRS.Events.GetEventStore`: Implementation for EventStore.&#x20;
  * Read more: [https://geteventstore.com/](https://geteventstore.com)

Recommended implementations:

* SQL Databases

The `Squidex.Infrastructure.CQRS.Events.IEventConsumerInfoRepository` defines the contract for a system to store the current state of each event consumer.

#### IAssetThumbnailGenerator

The `Squidex.Infrastructure.Assets.IAssetThumbnailGenerator` interface encapsulates image transformations. We only have an implementation for ImageSharp: [https://github.com/SixLabors/ImageSharp](https://github.com/SixLabors/ImageSharp)

### Repositories

You can provide other implementations for repositories, e.g. for Elastic Search or SQL:

* `Squidex.Domain.Apps.Entities.Assets.Repositories.IAssetRepository`: Stores metadata about assets such as names and sizes, but not the content itself. Can be challenging to implement the filtering.
* `Squidex.Domain.Apps.Entities.Contents.Repositoriess.IContentRepository`: Stores the content itself. Can be challenging to implement the filtering.
* `Squidex.Domain.Apps.Read.History.Repositories.IHistoryEventRepository`: Stores basic history events to show them in the UI.
* `Squidex.Domain.Apps.Entities.Rules.Repositories.IRuleEventRepository`: Stores the rule events, like an internal job queue.
* `Squidex.Infrastructure.States.IStore`: Key value store for json objects. Contains everything else like comments, apps, schemas, rules, custom indices and settings.
* `Squidex.Infrastructure.UsageTracking.IUsageTracking`: Stores historic usage information and performance metrics.

### Command Middleware&#x20;

Command middlewares are used to handle commands, for example when a new content item is created. They run in a pipeline and can be used to for a lot of different purposes. It is safe to implement the `ICustomCommandMiddleware` interface, that ensures that your commands are running in the correct order.

```csharp
namespace Squidex.Infrastructure.CQRS.Commands
{
    public interface ICustomCommandMiddleware
    {
        Task HandleAsync(CommandContext context, Func<Task> next);
    }
}
```

They accept two parameters. The first is the command context, that also includes a reference to the command. The next is a function to call the next command handler. Typical use cases are changes to one domain object, for example default fields for new schemas.

#### Example 1: Handle command

If you can accept the command, handle it and call `Complete()`.

```csharp
class MyHandler : ICustomCommandMiddleware
{
    public async Task HandleAsync(CommandContext context, Func<Task> next) 
    {
        if (context.Command is MyCommand myCommand)
        {
            await Handle(myCommand);
            context.Complete();
        }
        else
        {
            await next();
        }
    }
}
```

#### Example 2: Measure Performance

```csharp
class MyHandler : ICustomCommandMiddleware
{
    public async Task HandleAsync(CommandContext context, Func<Task> next) 
    {
        var watch = Stopwatch.StartNew();

        try
        {
            await next();
        }
        finally
        {
            watch.Stop();

            Log(watch);
        }
    }
}
```

#### Example 3: Enrich Command

```csharp
class MyHandler : ICustomCommandMiddleware
{
    public async Task HandleAsync(CommandContext context, Func<Task> next) 
    {
        if (context.Command is UserCommand userCommand)
        {
            userCommand.UserId = await GetUserIdAsync();
        }

        await next();
    }
}
```

### Event Consumers

Event consumers are invoked when new events are created or when an event consumer is restarted and old events are replayed. You should not raise new events, but of course you can create a new events.

```csharp
namespace Squidex.Infrastructure.CQRS.Events
{
    public interface IEventConsumer
    {
        // The name of the event consumer to display in the UI.
        string Name { get; }

        // Filter events by the stream name. Use regular expressions.
        string EventsFilter { get; }

        // Implement this method if you are interested to filter events before they are deserialized.
        bool Handles(StoredEvent @event);

        // Will be invoked when the event consumer is restarted.
        Task ClearAsync();

        // Will be called for each new or replayed event.
        Task On(Envelope<IEvent> @event);
    }
}
```
