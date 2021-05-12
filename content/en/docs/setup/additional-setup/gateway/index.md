---
title: Installing Gateways
description: Install and customize Istio Gateways.
weight: 40
keywords: [install,gateway,kubernetes]
owner: istio/wg-environments-maintainers
test: no
---

Along with creating a service mesh, Istio allows managing [gateways](/docs/concepts/traffic-management/#gateways),
which run at the edge of the mesh and allow fine-grained control over traffic entering and leaving the mesh.

Follow this guide to deploy one or more gateways.

## Prerequisites

This guide requires the Istio control plane [to be installed](/docs/setup/install/) before proceeding.

## Installing the gateway with injection

To support users that already utilize existing deployment tools, we provide a few different ways to deploy a gateway.
Each method will result in the same outcome; we recommend choosing the method you are most familiar with.
If you are not sure which to choose, we suggest the [Simple YAML](/docs/setup/additional-setup/gateway/#tabset-docs-setup-additional-setup-gateway-1-0-tab) method.

As a security best practice, it is recommended to deploy the gateway in a different namespace from the control plane.

Using the same mechanisms as [Istio sidecar injection](/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection), Istio allows the required
options for gateways to run to be automatically injected.
This enables developers to have full control over the gateway deployment, while simplifying operations - when a new upgrade is available, or a configuration has changed, gateway pods can simply be restarted to be updated.
This makes the experience of operating a gateway deployment the same as operating sidecars.

{{< tabset category-name="gateway-install-type" >}}

{{< tab name="Simple YAML" category-value="yaml" >}}

First, setup the Kubernetes configuration, called `ingress.yaml` here:

{{< text yaml >}}
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
  namespace: istio-ingress
spec:
  type: LoadBalancer
  selector:
    istio: ingressgateway
  ports:
  - port: 80
    name: http
  - port: 443
    name: https
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: istio-ingressgateway
  namespace: istio-ingress
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  template:
    metadata:
      annotations:
        # Enable gateway injection
        inject.istio.io/templates: gateway
      labels:
        # Set a unique label for the gateway. This is required to ensure Gateways
        # can select this workload
        istio: ingressgateway
        # Enable gateway injection. If connecting to a revisioned control plane, replace with
        # istio.io/rev: revision-name
        sidecar.istio.io/inject: "true"
    spec:
      containers:
      - name: istio-proxy
        image: auto # The image will automatically update each time the pod starts.
---
# Set up roles to allow reading credentials for TLS
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: istio-ingressgateway-sds
  namespace: istio-ingress
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: istio-ingressgateway-sds
  namespace: istio-ingress
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: istio-ingressgateway-sds
subjects:
- kind: ServiceAccount
  name: default
{{< /text >}}

Next, apply it to the cluster:

{{< text bash >}}
$ kubectl create namespace istio-ingress
$ kubectl apply -f ingress.yaml
{{< /text >}}

{{< tip >}}
The example above enables injection on the pod. This can also be controlled at the namespace level.
See [Controlling the injection policy](/docs/setup/additional-setup/sidecar-injection/#controlling-the-injection-policy) for more information.
{{< /tip >}}

{{< /tab >}}
{{< tab name="IstioOperator" category-value="iop" >}}

A call to `istioctl install` with [default settings](/docs/setup/install/istioctl/#install-istio-using-the-default-profile) will deploy a gateway by default.
However, this couples it to the control plane, making management and upgrade more complicated.
It is highly recommended to decouple these and allow independent operation.
This can be done by selecting the `minimal` profile, for example `istioctl install --set profile=minimal`.

First, setup an `IstioOperator` configuration file, called `ingress.yaml` here:

{{< text yaml >}}
apiVersion: operator.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: ingress
spec:
  profile: empty # Do not install CRDs or the control plane
  components:
    ingressGateways:
    - name: ingressgateway
      namespace: istio-ingress
      enabled: true
      label:
        # Set a unique label for the gateway. This is required to ensure Gateways
        # can select this workload
        istio: ingressgateway
  values:
    gateways:
      istio-ingressgateway:
        # Enable gateway injection
        injectionTemplate: gateway
{{< /text >}}

Then install using standard `istioctl` commands:

{{< text bash >}}
$ kubectl create namespace istio-ingress
$ istioctl install -f ingress.yaml
{{< /text >}}

{{< /tab >}}
{{< tab name="Helm" category-value="helm" >}}

First, set up a values configuration file, called `values.yaml` here:

{{< text yaml >}}
gateways:
  istio-ingressgateway:
    # Enable gateway injection
    injectionTemplate: gateway
    # Set a name for the gateway
    name: ingressgateway
    labels:
      # Set a unique label for the gateway. This is required to ensure Gateways
      # can select this workload
      istio: ingressgateway
{{< /text >}}

Then install using standard `helm` commands:

{{< text bash >}}
$ kubectl create namespace istio-ingress
$ helm install istio-ingress manifests/charts/gateways/istio-ingress -n istio-ingress -f values.yaml
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

## Managing gateways

The following describes how to manage gateways after installation. For more information on their usage, follow
the [Ingress](/docs/tasks/traffic-management/ingress/) and [Egress](/docs/tasks/traffic-management/egress/) tasks.

### Gateway selectors

The labels chosen on the gateway pods are used by Gateway selectors, so it is important that your
Gateway objects are configured to match these labels.

For example, in the deployments above we set the `istio=ingressgateway` label on the pods.
To apply a Gateway to these deployments, we would select the same label:

{{< text yaml >}}
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: gateway
spec:
  selector:
    istio: ingressgateway
...
{{< /text >}}

### Gateway deployment topologies

Depending on your mesh configuration and use cases, you may wish to deploy gateways in different ways.
Below shows a few different patterns for gateway deployments.
Note that multiple options can be utilized within the same cluster.

#### Shared gateway

In this model, a single centralized gateway is used by many applications, possibly across many namespaces.
Gateway(s) in the `ingress` namespace delegate ownership of routes to application namespaces, but retain control over TLS configuration.

{{< image width="50%" link="shared-gateway.svg" caption="Shared gateway" >}}

This model works well when you have many applications you want to expose externally, as they are able to use shared infrastructure.
It also works well in models that have the same domain or TLS certificates shared by many applications.

#### Dedicated application gateway

In this model, an application namespace has its own dedicated gateway installation.
This allows giving full control and ownership to a single namespace.
This level of isolation can be helpful for critical applications that have strict performance or security requirements.

{{< image width="50%" link="user-gateway.svg" caption="Dedicated application gateway" >}}

Unless there is another load balancer in front of Istio, this typically means that each application will have its own IP address,
which may complicate DNS configurations.

## Upgrading gateways

### In place upgrade

Because gateways utilize pod injection, new gateway pods that are created will automatically be injected with the latest configuration, which includes the version.

To pick up changes to the gateway configuration, the pods can simply be restarted, using commands such as `kubectl rollout restart deployment`.

If you would like to change the [control plane revision](/docs/setup/upgrade/canary/) in use by the gateway, you can set the `istio.io/rev` label on the gateway Deployment, which will also trigger a rolling restart.

{{< image width="50%" link="inplace-upgrade.svg" caption="In place upgrade in progress" >}}

### Canary upgrade

This upgrade method depends on using [control plane revisions](/docs/setup/upgrade/canary/).

If you would like to more slowly control a rollout of a new revision, we can run multiple versions of the gateway deployment.
For example, if I want to do a controlled rollout of the `canary` revision, create a copy of your current deployment with the `istio.io/rev=canary` label set:

{{< text yaml >}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: istio-ingressgateway-canary
  namespace: istio-ingress
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  template:
    metadata:
      annotations:
        inject.istio.io/templates: gateway
      labels:
        istio: ingressgateway
        istio.io/rev: canary # Set to the control plane revision you want to deploy
    spec:
      containers:
      - name: istio-proxy
        image: auto
{{< /text >}}

When this is created, we will now have two versions of the gateway, both selected by the same Service:

{{< text bash >}}
$ kubectl get endpoints -o "custom-columns=NAME:.metadata.name,PODS:.subsets[*].addresses[*].targetRef.name"
NAME                   PODS
istio-ingressgateway   istio-ingressgateway-788854c955-8gv96,istio-ingressgateway-canary-b78944cbd-mq2qf
{{< /text >}}

{{< image width="50%" link="canary-upgrade.svg" caption="Canary upgrade in progress" >}}

Unlike when deploying your own applications, you cannot utilize [Istio traffic shifting](/docs/tasks/traffic-management/traffic-shifting/) to control, as the traffic to the gateway is coming from external clients that we do not control.
Instead, you can control the distribution of traffic by the number of replicas of each deployment.
If you use another load balancer in front of Istio, you may also use that to control the traffic distribution.

{{< warning >}}
Because other installation methods bundle the gateway Service, which controls its external IP address, with the gateway Deployment,
only the [Simple YAML](/docs/setup/additional-setup/gateway/#tabset-docs-setup-additional-setup-gateway-1-0-tab) method is supported for this upgrade method.
{{< /warning >}}