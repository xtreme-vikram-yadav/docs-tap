# Scale workloads 

This topic describes the best practices required to build and deploy workloads at scale.

## Sample application reference

The following sections describe the configuration of the different-size applications used to derive
scalability best practices.

### Small

This is the simplest configuration and consists of the following services and workloads:

- API Gateway workload
- Search workload (In memory database)
- Search Processor workload
- Availability workload (In memory database)
- UI workload
- 3 Node RabbitMQ Cluster

### Medium

This includes all of the services of the small-size application and the following services and workloads:

- Notify workload
- Persistent Database (MySQL or Postgres)
- RabbitMQ Event Source
- KNative Eventing broker
- KNative triggers

### Large

This includes all of the services of the medium size application and the following services and workloads:

- Crawler Service
- Redis
- RabbitMQ backed eventing broker
- RabbitMQ backed triggers

## Application Configuration

The following section describes the application configuration used to derive the
scalability best practices.

**Supply chains used:**

- Out of the Box Supply Chain with Testing and Scanning (Build+Run)
- Out of the Box Supply Chain Basic +  Out of the Box Supply Chain with Testing (Iterate)

**Workload type**: `web`, `server` + `worker`

**Kubernetes Distribution**: Azure Kubernetes Service

**Number of applications deployed concurrently**: 50-55

|  | **CPU** |**Memory Range**| **Workload CRs in Iterate** |**Workload CRs in Build+Run**|**Workload Transactions per second**|
|:--- |:--- |:--- |:--- |:--- |:--- |
|**Small** | 500m - 700m |3-5 GB| 4 |5| 4 |
|**Medium** | 700m - 1000m |4-6 GB| NA |6|4 |
|**Large** | 1000m - 1500m |6-8 GB| NA |7|4 |

## Scale Configuration for workload deployments (Yet to arrive for updates)

Node configuration: 4 vCPUs, 16GB RAM, 120 GB Disk size

|**Cluster Type / Workload Details** |**Shared Iterate Cluster** | **Build Cluster** |**Run Cluster 1** |**Run Cluster 2**| **Run Cluster 3** |
|:--- |:--- |:--- |:--- |:---|:--- |:--- |
|**No. of Namespaces** |300| 333 | 333 | 333 | 333 |
|**Small** | 300 | 233 | 233 | 233 | 233 |
|**Medium** | | 83 | 83 | 83 | 83 |
|**Large** | | 17 | 17 | 17 | 17 |
|**No. of Nodes** |90 | 60 | 135 | 135 | 135 |

## Best Practices

The following table describes the resource limit changes that are required for components to support the scale configuration described in the previous table.

|**Controller/Pod**|**CPU Requests/Limits**|**Memory Requests/Limits**|**Other changes**|**Build** | **Run** | **Iterate** |**Changes made in**|
|:------|:------|:--------|:-------|:------|:------|:-----|:------|:--------|:-------|
 Build Service/kpack controller | 20&nbsp;m/100&nbsp;m | **1&nbsp;Gi/2&nbsp;Gi** || Yes | No | Yes | tap-values |
| Scanning/scan-link | 200&nbsp;m/500&nbsp;m | **1&nbsp;Gi/3&nbsp;Gi**| | Yes | No | No | tap-values |
| Cartographer| **3000&nbsp;m/4000&nbsp;m** | **10&nbsp;Gi/10&nbsp;Gi** | Concurrency 25 | Yes| Partial (only CPU) | Yes  | tap-values |
| Cartographer conventions| 100&nbsp;m/100&nbsp;m | 20&nbsp;Mi/**1.8&nbsp;Gi**  | 950&nbsp;Mi for concurrency - 25| Yes | Yes | Yes | tap-values |
| Namespace Provisioner | 100&nbsp;m/500&nbsp;m | **500&nbsp;Mi/2&nbsp;Gi** | | Yes | Yes | Yes | tap-values |
| Cnrs/knative-controller  | 100&nbsp;m/1000&nbsp;m | **512&nbsp;Mi/2&nbsp;Gi** | | No | Yes | Yes | overlay |
| Cnrs/net-contour | 40&nbsp;m/400&nbsp;m | **512&nbsp;Mi/2&nbsp;Gi** | Daemonset changed to Deployment with 3 replicas (set via tap-values) | No | Yes | Yes | overlay |
| Cnrs/activator | 300&nbsp;m/1000&nbsp;m | **5&nbsp;Gi/5&nbsp;Gi** |  | No | Yes | No | overlay |
| Cnrs/autoscaler  | 100&nbsp;m/1000&nbsp;m | **2&nbsp;Gi/2&nbsp;Gi** |  | No | Yes | No | overlay |
| Eventing/triggermesh | 50&nbsp;m/200&nbsp;m | **100&nbsp;Mi - 800&nbsp;Mi** | | No | Yes | Yes| overlay |
| tap-telemetry/tap-telemetry-informer | 100&nbsp;m/1000&nbsp;m | 100&nbsp;m/**2&nbsp;Gi** | | Yes| No | Yes| tap-values |

- CPU is measured in millicores. m = millicore. 1000 millicores = 1 vCPU.
- Memory is measured in Mebibyte and Gibibyte. Mi = Mebibyte. Gi = Gibibyte
- In the CPU Requests/Limits column, the changed values are bolded. Non bolded values in are the default ones set during a Tanzu Application Platform installation.
- In the CPU Requests/Limits column, some of the request and limits values are set equally so that the pod is allocated in a node where the requested limit is available.

\* Only when there is issue with scan pods getting deleted before Cartographer can process it

## Example resource limit changes

The following section provides examples of the changes required to the default limits to achieve scalability:

### Cartographer

The default Cartographer concurrency limits are:

```console
cartographer:
  cartographer:
    concurrency:
      max_workloads: 2
      max_deliveries: 2
      max_runnables: 2
```

Edit `values.yaml` to scale Cartographer concurrency limits where node configuration is 4 vCPUs, 16GB RAM, 120 GB Disk size:

```console
cartographer:
  cartographer:
    concurrency:
      max_workloads: 25
      max_deliveries: 25
      max_runnables: 25
```

The default resource limits are:

```console
resources:
  limits:
    cpu: 1
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi
```

Edit `values.yaml` to scale resource limit:

```console
# build-cluster
cartographer:
  cartographer:
    resources:
      limits:
        cpu: 4
        memory: 10Gi
      requests:
        cpu: 3
        memory: 10Gi

# run-cluster
cartographer:
  cartographer:
    resources:
      limits:
        cpu: 4
        memory: 2Gi
      requests:
        cpu: 3
        memory: 1G
```

### Cartographer-conventions

The default resource limits are:

```console
resources:
  limits:
    cpu: 100m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 20Mi
```

Edit `values.yaml` to scale resource limit:

```console
cartographer:
  conventions:
    resources:
      limits:
        memory: 1.8Gi
```

### Scan-link-controller

The default resource limits are:

```console
resources:
  limits:
    cpu: 250m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

Edit `values.yaml` to scale resource limit:

```console
scanning:
  resources:
    limits:
      cpu: 500m
      memory: 3Gi
    requests:
      cpu: 200m
      memory: 1Gi
```

### kpack-controller in build service

The default resource limits are:

```console
resources:
  limits:
    memory: 1Gi
  requests:
    cpu: 20m
    memory: 1Gi
```

Edit `values.yaml` to scale resource limits:

```console
buildservice:
  controller:
    resources:
      limits:
         memory: 2Gi
         cpu: 100m
      requests:
         memory: 1Gi
         cpu: 20m
```

### Namespace Provisioner

The default resource limits are:

```console
resources:
  limits:
    cpu: 500m
    memory: 100Mi
  requests:
    cpu: 100m
    memory: 20Mi
```

Edit `values.yaml` to scale resource limits:

```console
namespace_provisioner:
  controller_resources:
    resources:
      limits:
        cpu: 500m
        memory: 2Gi
      requests:
        cpu: 100m
        memory: 500Mi
```

### CNRS Knative Serving

The default resource limits are:

```console
resources:
  limits:
    cpu: 1
    memory: 1000Mi
  requests:
    cpu: 100m
    memory: 100Mi
```

Use an overlay to make the following resource limit changes:

```console
apiVersion: v1
kind: Secret
metadata:
  name: cnrs-patch-knative-controller
  namespace: tap-install
stringData:
  cnrs-patch-knative-controller.yaml: |
   #@ load("@ytt:overlay", "overlay")
   #@overlay/match by=overlay.subset({"kind":"Deployment", "metadata":{"name":"controller", "namespace": "knative-serving"}})
   ---
    spec:
      template:
        spec:
          containers:
            #@overlay/match by="name"
            - name: controller
              resources:
                limits:
                  cpu: 1
                  memory: 2Gi
                requests:
                  cpu: 100m
                  memory: 512Mi
```

### net-contour controller

The deployment type to be changed from Daemonset to Deployment.

```console
contour:
  envoy:
    workload:
      type: Deployment
      replicas: 3
```

The default resource limits are:

```console
resources:
  limits:
    cpu: 400m
    memory: 400Mi
  requests:
    cpu: 40m
    memory: 40Mi
```

Use an overlay to make the following resource limit changes:

```console
apiVersion: v1
kind: Secret
metadata:
  name: cnrs-patch-net-contour
  namespace: tap-install
stringData:
  cnrs-patch-net-contour.yaml: |
   #@ load("@ytt:overlay", "overlay")
   #@overlay/match by=overlay.subset({"kind":"Deployment", "metadata":{"name":"net-contour-controller", "namespace": "knative-serving"}})
   ---
    spec:
      template:
        spec:
          containers:
            #@overlay/match by="name"
            - name: controller
              resources:
                limits:
                  cpu: 400m
                  memory: 2Gi
                requests:
                  cpu: 40m
                  memory: 512Mi
```

### Autoscaler

The default resource limits are:

```console
resources:
  limits:
    cpu: 1
    memory: 1000Mi
  requests:
    cpu: 100m
    memory: 100Mi
```

Use an overlay to make the following resource limit changes:

```console
apiVersion: v1
kind: Secret
metadata:
  name: cnrs-patch-autoscaler
  namespace: tap-install
stringData:
  cnrs-patch-autoscaler.yaml: |
   #@ load("@ytt:overlay", "overlay")
   #@overlay/match by=overlay.subset({"kind":"Deployment", "metadata":{"name":"autoscaler", "namespace": "knative-serving"}})
   ---
    spec:
      template:
        spec:
          containers:
            #@overlay/match by="name"
            - name: autoscaler
              resources:
                limits:
                  cpu: 1000m
                  memory: 2Gi
                requests:
                  cpu: 100m
                  memory: 2Gi
```

### Activator

The default resource limits are:

```console
resources:
  limits:
    cpu: 1
    memory: 600Mi
  requests:
    cpu: 300m
    memory: 60Mi
```

Use an overlay to make the following resource limit changes:

```console
apiVersion: v1
kind: Secret
metadata:
  name: cnrs-patch-activator
  namespace: tap-install
stringData:
  cnrs-patch-activator.yaml: |
   #@ load("@ytt:overlay", "overlay")
   #@overlay/match by=overlay.subset({"kind":"Deployment", "metadata":{"name":"activator", "namespace": "knative-serving"}})
   ---
    spec:
      template:
        spec:
          containers:
            #@overlay/match by="name"
            - name: activator
              resources:
                limits:
                  cpu: 1000m
                  memory: 5Gi
                requests:
                  cpu: 300m
                  memory: 5Gi
```

### Triggermesh:

The default resource limits are:

```console
resources:
  limits:
    cpu: 200m
    memory: 300Mi
  requests:
    cpu: 50m
    memory: 100Mi
```

Use an overlay to make the following resource limit changes:

```console
apiVersion: v1
kind: Secret
metadata:
  name: triggermesh-patch-triggermesh-controller
  namespace: tap-install
stringData:
  triggermesh-patch-triggermesh-controller.yaml: |
   #@ load("@ytt:overlay", "overlay")
   #@overlay/match by=overlay.subset({"kind":"Deployment", "metadata":{"name":"triggermesh-controller", "namespace": "triggermesh"}})
   ---
   spec:
     template:
       spec:
         containers:
           #@overlay/match by="name"
           - name: controller
             resources:
               limits:
                 cpu: 200m
                 memory: 800Mi
               requests:
                 cpu: 50m
                 memory: 100Mi
```

### Tap Telemetry

The default resource limits are:

```console
resources:
  limits:
    cpu: 1
    memory: 1000Mi
  requests:
    cpu: 100m
    memory: 100Mi
```

Edit `values.yaml` to scale resource limits:

```console
tap_telemetry:
  limit_memory: 2Gi
```
