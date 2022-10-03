---
type: docs
title: "Using gRPC-based Pluggable Components"
linkTitle: "Using gRPC-based Pluggable Components"
weight: 250
description: "Extending dapr with external gRPC-based components"
---

For more information on how to develop gRPC-based Pluggable Components access the [Developing a gRPC-based Pluggable Component]({{< ref developing-grpc-components.md >}}).

The example above shows how a state store gRPC-based pluggable component is declared and used, the same can be applied for the others components.

{{< tabs "Standalone" "Kubernetes" >}}

{{% codetab %}}

## Step 1: Running the component

Your component should be up and running before starting dapr (i.e the Unix Socket must be created first). This is a requirement for standalone mode.

## Step 2: Declaring a gRPC-based Pluggable Component

gRPC-based Pluggable Componets are defined using the [Component CRD]({{< ref component-schema.md >}}) and its `type` is derived from the socket name (without the file extension).

Place the following file in the defined components-path (replace your_socket_goes_here by your component socket name without any extension).

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

> Note: 1.9.0 is the minimum dapr version that supports gRPC-based Pluggable Components
> specify the runtime version by using `--runtime-version` flag

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

As a prerequisite for running on kubernetes mode, your component must run as a container. Which means that it should be published first and accessible by your kubernetes cluster.

## Step 1: Deploying Dapr on a Kubernetes cluster

Follow the steps provided in the [Deploy Dapr on a Kubernetes cluster]({{< ref kubernetes-deploy.md >}}) docs and make sure you have a kubernetes cluster configured with dapr version 1.9+.

## Step 2: Adding Pluggable Component Container in your Deployments

When running on kubernetes mode, gRPC-based Pluggable Components are side-car containers.

As gRPC-based Pluggable Components are backed by Unix Domain Sockets, the first step is configuring the deployment spec to ensure that the socket created by your Pluggable Component are accessible by dapr runtime, to do that, you have to mount volumes and hint dapr where is the mounted unix socket volume and also, attach such volume to your Pluggable Component container.

Below you can see an example of a Deployment that configures a Pluggable Component:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  labels:
    app: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
      annotations:
        dapr.io/unix-domain-socket-path: "/tmp/dapr-components-sockets" ## required, the default path where dapr will discovery components.
        dapr.io/app-id: "my-app"
        dapr.io/enabled: "true"
        dapr.io/sidecar-listen-addresses: "0.0.0.0"
    spec:
      volumes: ## required, the sockets volume
        - name: dapr-unix-domain-socket
          emptyDir: {}
      containers:
        ### --------------------- YOUR APPLICATION CONTAINER GOES HERE -----------
        ##
        ### --------------------- YOUR APPLICATION CONTAINER GOES HERE -----------
        ### This is the pluggable component container.
        - name: component
          volumeMounts: # required, the sockets volume mount
            - name: dapr-unix-domain-socket
              mountPath: /dapr-unix-domain-sockets
          image: YOUR_IMAGE_GOES_HERE:YOUR_IMAGE_VERSION
          env:
            - name: DAPR_COMPONENTS_SOCKETS_FOLDER # Tells the component where the sockets should be created.
              value: /dapr-unix-domain-sockets
```

Great, do not apply the Deployment yet, let's add one more configuration: The Component CRD.

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
scopes:
  - backend
```

You should also [scope](https://docs.dapr.io/operations/components/component-scopes/) your component to make sure that only the target configured application will try to connect with the pluggable component since it will only be running in its Deployment, otherwise the runtime will fail when initializing the component.

> Note: Replace your_socket_goes_here by your component socket name without any extension

That's it! **[Apply the created manifests to your kubernetes cluster](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-apply)**, and then you are able to call the statestore APIs via Dapr API,

> Use [kubernetes pod forwarder](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) to access the daprd runtime.

See it working by running

```shell
curl -X POST -H "Content-Type: application/json" -d '[{ "key": "name", "value": "Bruce Wayne", "metadata": {}}]' http://localhost:$PORT/v1.0/state/prod-mystore
```

Retrieve the value

```shell
curl http://localhost:$PORT/v1.0/state/prod-mystore/name
```

> replace $PORT by dapr http port
> {{% /codetab %}}

{{< /tabs >}}
