# Canary Deployment Strategy
A canary rollout is a deployment strategy where the operator releases a new version of their application to a small percentage of the production traffic. 

## Overview
Since there is no agreed upon standard for a canary deployment, the rollouts controller allows users to outline how they want to run their canary deployment. Users can define a list of steps the controller uses to manipulate the ReplicaSets where there is a change to the `.spec.template`. Each step will be evaluated before the new ReplicaSet is promoted to the stable version, and the old version is completely scaled down.

Each step can have one of two fields. The `setWeight` field dictates the percentage of traffic that should be sent to the canary, and the `pause` struct instructs the rollout to pause.  When the controller reaches a `pause` step for a rollout, it will set adds a PauseCondition struct to the `.status.PauseConditions` field. If the `duration` field within the `pause` struct is set, the rollout will not progress to the next step until it has waited for the value of the `duration` field. Otherwise, the rollout will wait indefinitely until that Pause condition is removed. By using the `setWeight` and the `pause` fields, a user can declarative describe how they want to progress to the new version. Below is an example of a canary strategy.

!!! important
    If the canary Rollout does not use [traffic management](traffic-management/index.md), the Rollout makes a best effort attempt to achieve the percentage listed in the last `setWeight` step between the new and old version. For example, if a Rollout has 10 Replicas and 10% for the first `setWeight` step, the controller will scale the new desired ReplicaSet to 1 replicas and the old stable ReplicaSet to 9. In the case where the setWeight is 15%, the Rollout attempts to get there by rounding up the calculation (i.e. the new ReplicaSet has 2 pod since 15% * 10 rounds up to 2 and the old ReplicaSet has 9 pod since 85% * 10 rounds up to 9). If a user wants to have more fine-grained control of the percentages without a large number of Replicas, that user should use the  [traffic management](#trafficrouting) functionality.

## Example
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: example-rollout
spec:
  replicas: 10
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.4
        ports:
        - containerPort: 80
  minReadySeconds: 30
  revisionHistoryLimit: 3
  strategy:
    canary: #Indicates that the rollout should use the Canary strategy
      maxSurge: "25%"
      maxUnavailable: 0
      steps:
      - setWeight: 10
      - pause:
          duration: 1h # 1 hour
      - setWeight: 20
      - pause: {} # pause indefinitely
```

## Pause Duration
Pause duration can be specified with an optional time unit suffix. Valid time units are "s", "m", "h". Defaults to "s" if not specified.

```yaml
spec:
  strategy:
    canary:
      steps:
        - pause: { duration: 10 }  # 10 seconds
        - pause: { duration: 10s } # 10 seconds
        - pause: { duration: 10m } # 10 minutes
        - pause: { duration: 10h } # 10 hours
        - pause: {}                # pause indefinitely
```

If no `duration` specified for a pause step the rollout will be paused indefinitely. To unpause use the [argo kubectl plugin](kubectl-plugin.md) `promote` command. 

```shell
# promote to the next step
kubectl argo rollouts promote <rollout>
```

## Mimicking Rolling Update
If the steps field is omitted, the canary strategy will mimic the rolling update behavior. Similar to the deployment, the canary strategy has the `maxSurge` and `maxUnavailable` fields to configure how the Rollout should progress to the new version.


## Ephemeral Metadata

!!! important
    Available since v0.10.0

One use case is for a Rollout to label or annotate the canary/stable pods with user-defined labels/annotations,
for *only* the duration which they are the canary or stable set, and for the labels to be updated/removed as soon
as the ReplicaSet switches roles (e.g. from canary to stable). The use case which this enables, is to allow
prometheus, wavefront, datadog queries and dashboards to be built, which can rely on a consistent
labels, rather than the `rollouts-pod-template-hash` which is unpredictable and changing from revision
to revision.

A Rollout using the canary strategy has the ability to attach ephemeral metadata to the stable or 
canary Pods using the `stableMetadata` and `canaryMetadata` fields respectively.

```yaml
spec:
  strategy:
    canary:
      stableMetadata:
        labels:
          role: stable
      canaryMetadata:
        labels:
          role: canary
```

During an update, the Rollout will create the canary ReplicaSet while also merging the metadata
defined in `canaryMetadata` to the canary ReplicaSet's `spec.template.metadata`. This results in all
Pods of the ReplicaSet being created with the canary metadata. When the rollout becomes fully
promoted, the canary ReplicaSet becomes the stable, and is updated to use the labels and annotations
under `stableMetadata`. The Pods of the ReplicaSet will then be updated *in place* to use the stable
metadata (without recreating the pods).

!!! important
    In order for tooling to take advantage of this feature, they would need to recognize the change
    in labels and/or annotations that happen *after* the Pod has already started. Not all tools may
    detect this.


## Other Configurable Features
Here are the optional fields that will modify the behavior of canary strategy:

```yaml
spec:
  strategy:
    canary:
      analysis: object
      antiAffinity: object
      canaryService: string
      stableService: string
      maxSurge: stringOrInt
      maxUnavailable: stringOrInt
      trafficRouting: object
```

### analysis
Configure the background [Analysis](analysis.md) to execute during the rollout. If the analysis is unsuccessful the rollout will be aborted.

Defaults to nil

### antiAffinity
Check out the [Anti Affinity document](anti-affinity/anti-affinity.md) document for more information.

Defaults to nil

### canaryService
`canaryService` references a Service that will be modified to send traffic to only the canary ReplicaSet. This allows users to only hit the canary ReplicaSet.

Defaults to an empty string

### stableService
`stableService` the name of a Service which selects pods with stable version and don't select any pods with canary version. This allows users to only hit the stable ReplicaSet.

Defaults to an empty string

### maxSurge
`maxSurge` defines the maximum number of replicas the rollout can create to move to the correct ratio set by the last setWeight. Max Surge can either be an integer or percentage as a string (i.e. "20%")

Defaults to "25%".

### maxUnavailable
The maximum number of pods that can be unavailable during the update. Value can be an absolute number (ex: 5) or a percentage of desired pods (ex: 10%). This can not be 0 if MaxSurge is 0.

Defaults to 0

### trafficRouting
The [traffic management](traffic-management/index.md) rules to apply to control the flow of traffic between the active and canary versions. If not set, the default weighted pod replica based routing will be used.

Defaults to nil
