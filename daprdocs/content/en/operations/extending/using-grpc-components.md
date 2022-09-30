---
type: docs
title: "Using gRPC-based Pluggable Components"
linkTitle: "Using gRPC-based Pluggable Components"
weight: 250
description: "Extending dapr with external gRPC-based components"
---

For more information on how to develop gRPC-based Pluggable Components click [here]({{< ref developing-grpc-components.md >}}).

The example above shows how a state store gRPC-based pluggable component is declared and used, the same can be applied for the others components.

{{< tabs "Standalone" "Kubernetes" >}}

{{% codetab %}}

## Step 1: Running the component

Your component should be up and running before starting dapr (i.e the Unix Socket must be created first). This is a requirement for standalone mode.

## Step 2: Declaring a gRPC-based Pluggable Component

gRPC-based Pluggable Componets are defined using the [Component CRD]({{< ref component-schema.md >}}) and its `type` is derived from the socket name (without the file extension).

Place the following file in the defined components-path (replace your_socket_goes_here by our component socket name without any extension).

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: prod-mystore
spec:
  type: state.your_socket_goes_here
  version: v1
  metadata:
```

## Step 3: Running dapr

Init dapr by following the [tutorial]({{< ref get-started-api.md >}}), and make sure that your component CRD is placed in the right folder.

That's it! Now we are able to call the statestore APIs via Dapr API.

See it working by running

```shell
curl -X POST -H "Content-Type: application/json" -d '[{ "key": "name", "value": "Bruce Wayne", "metadata": {}}]' http://localhost:$PORT/v1.0/state/prod-mystore
```

Retrieve the value

```shell
curl http://localhost:$PORT/v1.0/state/prod-mystore/name
```

> replace $PORT by dapr http port

{{% /codetab %}}

{{% codetab %}}

## Step 1: Deploying the Component

## Step 2: Declaring a gRPC-based Pluggable Component

gRPC-based Pluggable Componets are defined using the [Component CRD]({{< ref component-schema.md >}}) and its `type` is derived from the socket name (without the file extension).

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: prod-mystore
spec:
  type: state.your_socket_goes_here
  version: v1
  metadata:
```

> Replace your_socket_goes_here by our component socket name without any extension

{{% /codetab %}}

{{< /tabs >}}
