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
| kind               |    Y     | The type of CRD. For Pluggable Components is must always be `PluggableComponent`                                           | `PluggableComponent` |
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

### Prerequisites

For this tutorial we assume that you have minimal knowledge of [gRPC and protocol buffers](https://grpc.io/), and that you choose a programming language [that supports gRPC](https://grpc.io/docs/languages/).

For simplicity, all code samples will use the generated code from the [protoc](https://developers.google.com/protocol-buffers/docs/downloads) tool.

As prior mentioned, dapr uses a Unix Domain Socket to communicate with Pluggable Components, which means that as a prerequisite you also need a UNIX-like system, or for windows users [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) should be sufficient.

### Generating gRPC code for the target language

The protobuf definitions for Pluggable Components reside in the [dapr](https://github.com/dapr/dapr) runtime repository, you must clone it first to get a copy of proto definitions in your machine.

{{< tabs SSH HTTP >}}
{{% codetab %}}

```shell
git clone git@github.com:dapr/dapr.git
```

{{% /codetab %}}

{{% codetab %}}

```shell
git clone https://github.com/dapr/dapr.git
```

{{% /codetab %}}

{{< /tabs >}}

> You can also [download as a zip file](https://github.com/dapr/dapr/archive/refs/heads/master.zip), and extract using the [unzip](https://linux.die.net/man/1/unzip) command.

Once you have the proto files in your machine, go to the repository root folder, copy out the proto directory [dapr/proto(https://github.com/dapr/dapr/tree/master/dapr/proto)] and follow your target language guide for them.

> Note that you only need the `components/v1` folder and the `common/v1` folder leave the rest untouched.

{{< tabs Java ".NET">}}

{{% codetab %}}
https://grpc.io/docs/languages/java/generated-code/#codegen
{{% /codetab %}}

{{% codetab %}}
https://learn.microsoft.com/en-us/aspnet/core/grpc/basics?view=aspnetcore-6.0#c-tooling-support-for-proto-files
{{% /codetab %}}

{{< /tabs >}}

### Implementing your desired Service

You can choose either, [StateStore](https://github.com/dapr/dapr/blob/9da16f4a2ee9658dc7b26cf154d75c2e7a3128b7/dapr/proto/components/v1/state.proto#L119), [PubSub](https://github.com/dapr/dapr/blob/9da16f4a2ee9658dc7b26cf154d75c2e7a3128b7/dapr/proto/components/v1/pubsub.proto#L23) or Bindings to implement your component, pick one of them and implement the desired service with your desired language.
