---
type: docs
title: "Pluggable Components"
linkTitle: "Developing Components"
weight: 250
description: "Extend dapr with external gRPC-based components"
---

Pluggable Components are [gRPC-based](https://grpc.io/) dapr components generally running as containers or processes communicating via [Unix Domain Sockets](https://en.wikipedia.org/wiki/Unix_domain_socket).

## Declaring

Pluggable Componets are defined using its own [CustomResourceDefinition](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) that are not part of the [Component CRD]({{< ref component-schema.md >}}). Instead, Pluggable Components are defined separately and referenced as `type` spec when declaring a dapr component.

```yaml
apiVersion: dapr.io/v1alpha1
kind: PluggableComponent
metadata:
  name: my-state-store
spec:
  componentName: custom-store
  type: state
  version: v1
```

| Field              | Required | Details                                                                                                                    | Example              |
| ------------------ | :------: | -------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| apiVersion         |    Y     | The version of the Dapr (and Kubernetes if applicable) API you are calling                                                 | `dapr.io/v1alpha1`   |
| kind               |    Y     | The type of CRD. For Pluggable Components it must always be `PluggableComponent`                                           | `PluggableComponent` |
| **metadata**       |    -     | **Information about the pluggable component registration**                                                                 |
| metadata.name      |    Y     | The name of the kubrnetes object                                                                                           | `my-state-store`     |
| metadata.namespace |    N     | The namespace for the pluggable component for hosting environments with namespaces                                         | `myapp-namespace`    |
| **spec**           |    -     | **Detailed information on the pluggable component resource**                                                               |
| spec.type          |    Y     | The type of the Pluggable Component, it should be one of the following: `state`, `pubsub`, `inputbinding`, `outputbinding` | `state`              |
| spec.version       |    Y     | The version of the Pluggable Component                                                                                     | `v1`                 |
| spec.componentName |    N     | The Component name that will be referenced. The metadata.Name is used when not specified.                                  | `custom-store`       |

the mentioned pluggable component will be referenced later with using the complete name of it: `state.custom-store` using the [Component CRD]({{< ref component-schema.md >}}).

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: prod-mystore
spec:
  type: state.custom-store
  version: v1
```

## Developing a Pluggable Component from scratch

{{< tabs ".NET" >}}
{{% codetab %}}

### Step 1: Prerequisites

For this tutorial we assume that you have minimal knowledge of [gRPC and protocol buffers](https://grpc.io/), and that you choose a programming language [that supports gRPC](https://grpc.io/docs/languages/).

For simplicity, all code samples will use the generated code from the [protoc](https://developers.google.com/protocol-buffers/docs/downloads) tool.

As prior mentioned, dapr uses a Unix Domain Socket to communicate with Pluggable Components, which means that as a prerequisite you also need a UNIX-like system, or for windows users [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) should be sufficient.

And also [.NET Core 6+](https://dotnet.microsoft.com/en-us/download)

This tutorial is based on the [official microsoft documentation](https://learn.microsoft.com/en-us/aspnet/core/grpc/basics?view=aspnetcore-6.0#c-tooling-support-for-proto-files).

### Step 2: Generating gRPC code for the target language

The protobuf definitions for Pluggable Components reside in the [dapr](https://github.com/dapr/dapr) runtime repository, you must clone it first to get a copy of proto definitions in your machine.

```shell
git clone https://github.com/dapr/dapr.git
```

> You can also [download as a zip file](https://github.com/dapr/dapr/archive/refs/heads/master.zip), and extract using the [unzip](https://linux.die.net/man/1/unzip) command.

Once you have the proto files in your machine, go to the repository root folder, copy out the proto directory [dapr/proto](https://github.com/dapr/dapr/tree/master/dapr/proto).

> Note that you only need the `components/v1` folder and the `common/v1` folder leave the rest untouched.

### Step 3: Implementing your desired Service

You can choose either, [StateStore](https://github.com/dapr/dapr/blob/9da16f4a2ee9658dc7b26cf154d75c2e7a3128b7/dapr/proto/components/v1/state.proto#L119), [PubSub](https://github.com/dapr/dapr/blob/9da16f4a2ee9658dc7b26cf154d75c2e7a3128b7/dapr/proto/components/v1/pubsub.proto#L23) or [Bindings](https://github.com/dapr/dapr/blob/master/dapr/proto/components/v1/bindings.proto#L1) to implement your component, pick one of them and implement the desired service using your desired language.

<img src="/images/pluggable-component-arch.png" width=800 alt="Diagram showing the final MemStore design">

Start with an empty folder and create a new .NET gRPC service by running:

```shell
dotnet new grpc -o DaprMemStoreComponent
```

Now our directory tree should be something like this:

```shell
➜  memstore-dotnet tree .
.
└── DaprMemStoreComponent
    ├── DaprMemStoreComponent.csproj
    ├── Program.cs
    ├── Properties
    │   └── launchSettings.json
    ├── Protos
    │   └── greet.proto
    ├── Services
    │   └── GreeterService.cs
    ├── appsettings.Development.json
    ├── appsettings.json
    └── obj
        ├── DaprMemStoreComponent.csproj.nuget.dgspec.json
        ├── DaprMemStoreComponent.csproj.nuget.g.props
        ├── DaprMemStoreComponent.csproj.nuget.g.targets
        ├── project.assets.json
        └── project.nuget.cache

5 directories, 12 files
```

Remove the file `Protos/greet.proto` and copy out the `dapr/proto/components/v1` and `dapr/proto/common/v1` folders from dapr repository to the `DaprMemStoreComponent/Protos` directory.

```shell
➜  memstore-dotnet tree .
.
└── DaprMemStoreComponent
    ├── DaprMemStoreComponent.csproj
    ├── Program.cs
    ├── Properties
    │   └── launchSettings.json
    ├── Protos
    │   └── dapr
    │       └── proto
    │           ├── common
    │           │   └── v1
    │           │       └── common.proto
    │           └── components
    │               └── v1
    │                   ├── bindings.proto
    │                   ├── common.proto
    │                   ├── pubsub.proto
    │                   └── state.proto
    ├── Services
    │   └── GreeterService.cs
    ├── appsettings.Development.json
    ├── appsettings.json
    └── obj
        ├── DaprMemStoreComponent.csproj.nuget.dgspec.json
        ├── DaprMemStoreComponent.csproj.nuget.g.props
        ├── DaprMemStoreComponent.csproj.nuget.g.targets
        ├── project.assets.json
        └── project.nuget.cache

11 directories, 16 files
```

Now, open the project file `DaprMemStoreComponent.csproj`, and find the live above

```xml
    <Protobuf Include="Protos\greet.proto" GrpcServices="Server" />
```

Remove that line, and add the following ones:

```xml
    <Protobuf Include="Protos\dapr\proto\components\v1\bindings.proto" ProtoRoot="Protos" GrpcServices="Client,Server" />
    <Protobuf Include="Protos\dapr\proto\components\v1\pubsub.proto" ProtoRoot="Protos" GrpcServices="Client,Server" />
    <Protobuf Include="Protos\dapr\proto\components\v1\state.proto" ProtoRoot="Protos" GrpcServices="Client,Server" />
    <Protobuf Include="Protos\dapr\proto\components\v1\shared.proto" ProtoRoot="Protos" GrpcServices="Client" />
    <Protobuf Include="Protos\dapr\proto\common\v1\common.proto" ProtoRoot="Protos" GrpcServices="Client" />
```

Now let's create our `MemStoreService`, remove the `GreeterService.cs` and create our `MemStoreService.cs` with the following content,

```csharp
using Dapr.Proto.Components.V1;

namespace DaprMemStoreComponent.Services;

public class MemStoreService : StateStore.StateStoreBase
{
    private readonly ILogger<MemStoreService> _logger;
    public MemStoreService(ILogger<MemStoreService> logger)
    {
        _logger = logger;
    }
}
```

Now, go to the `Program.cs` and find the line `app.MapGrpcService<GreeterService>();` and `GreeterService` by our `MemStoreService` the final result will be,

```csharp
using DaprMemStoreComponent.Services;
using Microsoft.AspNetCore.Server.Kestrel.Core;

var builder = WebApplication.CreateBuilder(args);

// Additional configuration is required to successfully run gRPC on macOS.
// For instructions on how to configure Kestrel and gRPC clients on macOS, visit https://go.microsoft.com/fwlink/?linkid=2099682

// Add services to the container.
builder.WebHost.ConfigureKestrel(options =>
            {
                options.ListenLocalhost(5000, o => o.Protocols = HttpProtocols.Http2); // necessary for mac users.
            });

builder.Services.AddGrpc();

var app = builder.Build();

// Configure the HTTP request pipeline.
app.MapGrpcService<MemStoreService>();
app.MapGet("/", () => "Communication with gRPC endpoints must be made through a gRPC client. To learn how to create a client, visit: https://go.microsoft.com/fwlink/?linkid=2086909");

app.Run();
```

{{% /codetab %}}

{{< /tabs >}}

# https://learn.microsoft.com/en-us/aspnet/core/grpc/basics?view=aspnetcore-6.0#c-tooling-support-for-proto-files
