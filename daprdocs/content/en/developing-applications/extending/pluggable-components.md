---
type: docs
title: "Developing Components"
linkTitle: "Developing Components"
weight: 250
description: "Extending dapr with external gRPC-based components"
---

[gRPC-based](https://grpc.io/) Dapr components are typically run as containers or processes that communicate with the dapr main process via [Unix Domain Sockets](https://en.wikipedia.org/wiki/Unix_domain_socket).

## Developing a Pluggable Component from Scratch

{{< tabs ".NET" >}}
{{% codetab %}}

### Step 1: Prerequisites

For this tutorial, we assume that you have minimal knowledge of [gRPC and protocol buffers](https://grpc.io/), and that you chose a programming language [that supports gRPC](https://grpc.io/docs/languages/).

For simplicity, all code samples will use the generated code from the [protoc](https://developers.google.com/protocol-buffers/docs/downloads) tool or supported gRPC build tools by languages.

As previously mentioned, dapr uses a Unix Domain Socket to communicate with gRPC-based components, which means that as a prerequisite you also need a UNIX-like system, or for Windows users, [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) should be sufficient.

Also, we are going to use [.NET Core 6+](https://dotnet.microsoft.com/en-us/download)

This tutorial is based on the [official Microsoft documentation](https://learn.microsoft.com/en-us/aspnet/core/grpc/basics?view=aspnetcore-6.0#c-tooling-support-for-proto-files).

### Step 2: Downloading the quickstart code

We've prepared a quickstart sample code that helps you to skip a few manual steps like cloning dapr repository and adding the required libraries. Get started by downloading and unzipping it using the following <a href="/code/dotnet-memstore.zip">link</a>.

This quickstart contains all the required bits to start developing a gRPC-based component from scratch, including protobuf definitions and an unimplemented InMemory StateStore that we're going to implement.

### Step 3: Building and running

At this point, we are ready to build and run our component. Let's test it by running `dotnet build`

```
➜  DaprMemStoreComponent dotnet build
MSBuild version 17.3.0+92e077650 for .NET
  Determining projects to restore...
  All projects are up-to-date for restore.
  DaprMemStoreComponent -> [redacted]

Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:01.03
```

Great!

Now, let's run the StateStore service by issuing `dotnet run`

```
➜  DaprMemStoreComponent dotnet run
Building...
warn: Microsoft.AspNetCore.Server.Kestrel[0]
      Overriding address(es) 'http://localhost:5259, https://localhost:7089'. Binding to endpoints defined via IConfiguration and/or UseKestrel() instead.
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://unix:/tmp/dapr-components-sockets/memstore.sock
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
info: Microsoft.Hosting.Lifetime[0]
```

To see it working, I'll use the [grpc_cli](https://github.com/grpc/grpc/blob/master/doc/command_line_tool.md) tool for making gRPC calls. You can download and use it or use your preferred tool.

Once properly downloaded and installed, open a new terminal, and let's now invoke the `Features` method from the StateStore service using the following command: `grpc_cli call unix:///tmp/dapr-components-sockets/memstore.sock Features ''`

```
➜ memstore-dotnet grpc_cli call unix:///tmp/dapr-components-sockets/memstore.sock Features ''
connecting to unix:///tmp/dapr-components-sockets/memstore.sock
Received trailing metadata from server:
content-length : 0
date : Thu, 29 Sep 2022 18:26:54 GMT
server : Kestrel
Rpc failed with status code 12, error message:
```

Great, don't worry about errors for now.

Now we are ready to go,

### Step 4: Implementing an In-Memory StateStore

<img src="/images/pluggable-component-arch.png" width=800 alt="Diagram showing the final MemStore design">

Go to our `MemStoreService.cs` and let's override the Features method.

```csharp
using Dapr.Client.Autogen.Grpc.v1;
using Dapr.Proto.Components.V1;
using Grpc.Core;

namespace DaprMemStoreComponent.Services;

public class MemStoreService : StateStore.StateStoreBase
{
    private readonly ILogger<MemStoreService> _logger;
    public MemStoreService(ILogger<MemStoreService> logger)
    {
        _logger = logger;
    }

    public override Task<FeaturesResponse> Features(FeaturesRequest request, ServerCallContext ctx)
    {
        return Task.FromResult(new FeaturesResponse { });
    }
}

```

Great, stop the current `dotnet run` execution and reissue the run command.

Let's make the same call as we did before: `grpc_cli call unix:///tmp/dapr-components-sockets/memstore.sock Features ''`

Now we should get our expected OK response;

```
➜ memstore-dotnet grpc_cli call unix:///tmp/dapr-components-sockets/memstore.sock Features ''
connecting to unix:///tmp/dapr-components-sockets/memstore.sock
Received initial metadata from server:
date : Fri, 30 Sep 2022 14:36:56 GMT
server : Kestrel
Rpc succeeded with OK status
```

As the goal here is to create a simple in-memory statestore, let's use a .NET `ConcurrentDictionary` as our persistence layer. Go again to the `MemStoreService.cs`

> Note: It should be static because the framework recreates our StateStore with each request.

```csharp
using System.Collections.Concurrent;
using Dapr.Client.Autogen.Grpc.v1;
using Dapr.Proto.Components.V1;
using Grpc.Core;

namespace DaprMemStoreComponent.Services;

public class MemStoreService : StateStore.StateStoreBase
{
    private readonly ILogger<MemStoreService> _logger;
    private readonly static IDictionary<string, string?> Storage = new ConcurrentDictionary<string, string?>();
    public MemStoreService(ILogger<MemStoreService> logger)
    {
        _logger = logger;
    }

    public override Task<FeaturesResponse> Features(FeaturesRequest request, ServerCallContext ctx)
    {
        return Task.FromResult(new FeaturesResponse { });
    }
}

```

Now let's override the Get and Set methods,

```csharp
using System.Collections.Concurrent;
using Dapr.Client.Autogen.Grpc.v1;
using Dapr.Proto.Components.V1;
using Google.Protobuf;
using Grpc.Core;

namespace DaprMemStoreComponent.Services;

public class MemStoreService : StateStore.StateStoreBase
{
    private readonly ILogger<MemStoreService> _logger;
    private readonly static IDictionary<string, string?> Storage = new ConcurrentDictionary<string, string?>();
    public MemStoreService(ILogger<MemStoreService> logger)
    {
        _logger = logger;
    }

    public override Task<FeaturesResponse> Features(FeaturesRequest request, ServerCallContext ctx)
    {
        return Task.FromResult(new FeaturesResponse { });
    }

    public override Task<GetResponse> Get(GetRequest request, ServerCallContext ctx)
    {
        if (Storage.TryGetValue(request.Key, out var data))
        {
            return Task.FromResult(new GetResponse
            {
                Data = ByteString.CopyFromUtf8(data),
            });
        }
        return Task.FromResult(new GetResponse { }); // in case of not found you should not return any error.
    }

    public override Task<SetResponse> Set(SetRequest request, ServerCallContext ctx)
    {
        Storage[request.Key] = request.Value?.ToStringUtf8();
        return Task.FromResult(new SetResponse());
    }
}
```

Great, let's re-run our service and try out a simple set call:

```shell
grpc_cli call unix:///tmp/dapr-components-sockets/memstore.sock dapr.proto.components.v1.StateStore/Set "key:'my_key', value:'my_value'"
```

You should receive an OK

```
➜ memstore-dotnet grpc_cli call unix:///tmp/dapr-components-sockets/memstore.sock dapr.proto.components.v1.StateStore/Set "key:'my_key', value:'my_value'"
connecting to unix:///tmp/dapr-components-sockets/memstore.sock
Received initial metadata from server:
date : Fri, 30 Sep 2022 14:49:38 GMT
server : Kestrel
Rpc succeeded with OK status
```

Now let's retrieve the value,

```shell
grpc_cli call unix:///tmp/dapr-components-sockets/memstore.sock dapr.proto.components.v1.StateStore/Get "key:'my_key'"
```

```
➜  memstore-dotnet grpc_cli call unix:///tmp/dapr-components-sockets/memstore.sock dapr.proto.components.v1.StateStore/Get "key:'my_key'"

connecting to unix:///tmp/dapr-components-sockets/memstore.sock
Received initial metadata from server:
date : Fri, 30 Sep 2022 15:36:00 GMT
server : Kestrel
data: "my_value"
Rpc succeeded with OK status
```

At this point our component is partially implemented, more methods like `bulk*` operations should be added to consider functionally complete, but there are two important methods that should be implemented to consider ready to be used by dapr, they are `Init` and `Ping` methods.

### Step 5: Init and Ping

The ping method will be used by dapr runtime as a liveness probe, so it's up to you to decide what is `liveness` from your component point of view. As a simple implementation `Ping` can just respond without any deep check, but it is a requirement to work with daprd.

```csharp
    public override Task<PingResponse> Ping(PingRequest request, ServerCallContext ctx)
    {
        return Task.FromResult(new PingResponse());
    }
```

Init method is called as part of dapr initialization, it will be called first before any interaction with others components methods. The component can use init to create database connections, make async calls or whatever is necessary to consider your component ready to work, be mindful on time consuming operations because there are timeouts associated to initializing components.

Init receives component metadata as a parameter and there are no semantics associated with it, metadata can be anything the component needs to be ready, like connection strings, timeouts or services addresses.

Our final MemStoreService will be:

```csharp
using System.Collections.Concurrent;
using Dapr.Client.Autogen.Grpc.v1;
using Dapr.Proto.Components.V1;
using Google.Protobuf;
using Grpc.Core;

namespace DaprMemStoreComponent.Services;

public class MemStoreService : StateStore.StateStoreBase
{
    private readonly ILogger<MemStoreService> _logger;
    private readonly static IDictionary<string, string?> Storage = new ConcurrentDictionary<string, string?>();
    public MemStoreService(ILogger<MemStoreService> logger)
    {
        _logger = logger;
    }

    public override Task<FeaturesResponse> Features(FeaturesRequest request, ServerCallContext ctx)
    {
        return Task.FromResult(new FeaturesResponse { });
    }

    public override Task<GetResponse> Get(GetRequest request, ServerCallContext ctx)
    {
        if (Storage.TryGetValue(request.Key, out var data))
        {
            return Task.FromResult(new GetResponse
            {
                Data = ByteString.CopyFromUtf8(data),
            });
        }
        return Task.FromResult(new GetResponse { }); // in case of not found you should not return any error.
    }

    public override Task<SetResponse> Set(SetRequest request, ServerCallContext ctx)
    {
        Storage[request.Key] = request.Value?.ToStringUtf8();
        return Task.FromResult(new SetResponse());
    }

    public override Task<InitResponse> Init(InitRequest request, ServerCallContext ctx)
    {
        return Task.FromResult(new InitResponse { });
    }

    public override Task<PingResponse> Ping(PingRequest request, ServerCallContext ctx)
    {
        return Task.FromResult(new PingResponse());
    }
}
```

> There are other missing methods like `bulk*` and `delete`, at this point you might be able to implement it without our help.

## Declaring

gRPC-based Pluggable Componets are defined using the [Component CRD]({{< ref component-schema.md >}}) and its `type` is derived from the socket name (without the file extension).

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: prod-mystore
spec:
  type: state.memstore
  version: v1
```

{{% /codetab %}}

{{< /tabs >}}
