---
kep-number: 0
title: Pod Termination Reason
authors:
  - "@kow3ns"
  - "@yguo0905"
  - "@krmayankk"
  - "@barney-s"
owning-sig: sig-apps
participating-sigs:
reviewers:
  - TBD
approvers:
  - TBD
editor: TBD
creation-date: 2018-03-19
last-updated: 2018-03-19
status: provisional
see-also:
replaces:
superseded-by:
---

# Pod Termination Hint

## Table of Contents

* [Table of Contents](#table-of-contents)
* [Summary](#summary)
* [Motivation](#motivation)
    * [Goals](#goals)
    * [Non-Goals](#non-goals)
* [Proposal](#proposal)
    * [User Stories](#user-stories)
    * [Implementation](#implementation)
    * [Risks and Mitigations](#risks-and-mitigations)
* [Graduation Criteria](#graduation-criteria)
* [Implementation History](#implementation-history)
* [Drawbacks](#drawbacks)
* [Alternatives](#alternatives)


## Summary

Currently, as of Kubernetes 1.9, when a Pod is [terminated](https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods),  the Pod's containers will have their [pre-stop handlers](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#hook-handler-execution) invoked prior to receiving a `TERM` signal. Kubelet will wait until the specified termination grace period has expired, and then send a `KILL` signal. The cleanup tasks performed are often dependent on the reason for termination. In this proposal, we design a mechanism to deliver a signal to a Pod's 
containers indicating why the Pod has been terminated. Applications can use 
this to determine what actions they should take during the termination.

## Motivation

Consider a stateful, replicated storage application where the application's data are non-uniformly replicated across servers. Such replication schemes are common for BASE storage applications (e.g. Cassandra, Riak) and for sharded, replicated, strongly consistent storage applications (e.g Couchbase, MongoDB, HBase). For many such applications, it is not atypical for a single server to be responsible for serving TiBs of data.
When a Pod containing such an application is terminated because it will be decommissioned, terminated without the intent of resurrection (e.g. when the user horizontally scales the application down), during its termination grace period, we expect the application, or a process managing the application, to do the following.

1. Drain all open client connections.
2. Flush any in memory data structures to the file system.
3. Synchronize the file system with storage media.
4. Trigger a global redistribution of data so that the application's data are not left under replicated.
5. Modify the application membership to remove the decommissioned instance.

However, in contrast to the above, when a Pod is terminated with the intention that it should be rescheduled (e.g. The Pod's controller is performing a configuration, image, or resource update, the Pod is being moved due to a taint, or the Pod is being evicted due to a managed node upgrade), we expect the application, or its managing process, to do the following.
Drain all open client connections.
Flush any in memory data structures to the file system.
Synchronize the file system with storage media.
Gracefully terminated the application without triggering a data redistribution.
This will leave the application temporarily under replicated, and, we claim that, for a distributed storage application this temporary under replication is desirable. Consider that, such applications are designed to remain available during some number of temporary process failures, and PodDisruptionBudgets provide a way for application administrators to communicate the number of planned temporary process failures (disruptions) that an application can tolerate.
When we expect a terminated Pod to be rescheduled immediately, or nearly so, redistributing its data will only increase the expected time to recovery. This is only exacerbated by the fact that when the terminated Pod is resurrected, the application will have to once again redistribute its data to make use of the restored capacity. For applications that may require TiBs of data to be redistributed for a single Pod update, the network, storage, and time costs associated with redistribution are prohibitive when there is a very high probability that we can reschedule the Pod immediately.
If the application's Persistent Volumes are backed by local storage media (note that this is not currently possible but we expect this feature to land in the near future), we expect a decommissioned instance to behave identically to an instance whose PVs are backed by remote storage media. However, as a local PV is intrinsically tied to the lifetime of the Node, leaving the application under replicated during destructive updates to the Pod is inherently more risky. We still believe that, under these circumstances, temporary under replication is a viable strategy, and users can mitigate their risk by increasing the number of replicas of the application's data.

### Goals

Provide an ability to inject reason when terminating pods
K8s control plane triggered pod deletion would indicate appropriate reason

### Non-Goals

Describing semantics of reason (except for k8s triggered pod deletions)
Implementing pre-delete hook or SIGTERM handler based on reason

## Proposal

### User Stories

#### Cassandra pod termination
When bringing down a pod permanently we would like to do a ‘nodetool decommission’ to redistribute the owned token ranges. But if we are deleting the pod due to eviction (would be recreated later by the statefulset), then we don’t need to do a decommission.

#### Detect intolerable taints
If a pod is being evicted due to intolerable taints and take necessary action.

#### Singal watchers on pod termination reason
As the administrator of a Kubernetes cluster, I want to provide a signal to tenant applications that they are being terminated in violation of their declared disruption budgets or termination grace periods due to a superseding, global, organization policy (e.g. My organization requires that an application's termination grace period and disruption budget allow managed upgrades to make progress.)

As a user I want to be able to send a termination reason to my Pods when I use the Kubernetes API or kubectl to gracefully delete a Pod.

As a Reliability Engineer for an application running on a Kubernetes cluster, I want to configure Pods to communicate the reason that they were terminated via a HTTP GET request so that I can configure monitoring and alerting based on historic trends.


### Implementation

#### Assumptions

1. Kubelet will always invoke the pre-stop handler prior to sending a TERM signal to the entry point of a container.
2. pre-stop handlers will not contain complicated or long running business logic. The logic associated with container cleanup will be executed by the container's entry point during the container's configured termination grace period.
3. Termination reason is a general string that can be associated with an ObjectMeta at that time of deletion. Currently, it only has semantic meaning with respect to the graceful deletion of individual Pods. We do not, at this time, propose to implement controller assisted propagation of termination reasons from one API object to another (e.g Setting a termination reason for a ReplicaSet which the controller will then propagate to its Pods). Such considerations will be deferred to future work.

#### Requirements

1. Termination reason must be available to a container at the time its pre-stop handler is invoked. In this way, applications can perform any actions necessary to configure the container's entry point prior to the entry point receiving the TERMsignal.
2. The delivery mechanism for termination reason must be consumable by, and compatible with, existing HTTP and command based pre-stop handlers.
3. Any headers, variables, or parameters used to communicate a termination reason must not be hard coded, and the receiving container must be able to override the defaults.

#### API Objects
Termination reason is implemented as a generic string that can be associated with the metadata of any object that supports graceful termination. Initially, the reason will only be actionable with respect to Pods' containers, but this generic delivery mechanism can be extended to other use cases.
DeleteOptions is modified to carry a reason.

```
type DeleteOptions struct {
	// Other fields omitted for brevity.
	
	// Reason indicates the reason for the deletion of an API Object that 
	// undergo graceful termination upon deletion.
	Reason string `json:"reason,omitempty"`
}
```

The ObjectMeta struct is modified to carry the deletion reason via an Object's metadata.
```
type ObjectMeta {
	// Other fields omitted for brevity.
	
    // DeletionReason indicates the reason for the deletion of an API Object. 
    // Its purpose is to provide an extra generic signal to watchers of API 
    // Objects during the graceful termination process. 
	DeletionReason string `json:"reason,omitempty"`
}
```

DeleteExecAction and DeleteHttpAction are introduced to provide users with the ability to configure the environment variable or HTTP Header that will be used to convey a termination reason signaled by a DeletionReason.
```
// DeleteExecAction describes a "run in container" action that will be invoked
// inside of a container prior to sending the TERM signal to the 
// container's entry point.
type DeleteExecAction struct {
	// Command is the command line to execute inside the container, the working directory for the
	// command  is root ('/') in the container's filesystem.  The command is simply exec'd, it is
	// not run inside a shell, so traditional shell instructions ('|', etc) won't work.  To use
	// a shell, you need to explicitly call out to that shell.
	// +optional
	Command []string
	
	// ReasonEnv is the environment variable that wil be populated with the 
	// reason, if provided, for the Pod's termination. This variable defaults
	// to "KUBE_DELETE_REASON"
	ReasonEnv string
}

// DeleteHTTPGetAction describes an action to take upon deletion based on HTTP 
// Get requests.
type DeleteHTTPGetAction struct {
	// Optional: Path to access on the HTTP server.
	// +optional
	Path string
	// Required: Name or number of the port to access on the container.
	// +optional
	Port intstr.IntOrString
	// Optional: Host name to connect to, defaults to the pod IP. You
	// probably want to set "Host" in httpHeaders instead.
	// +optional
	Host string
	// Optional: Scheme to use for connecting to the host, defaults to HTTP.
	// +optional
	Scheme URIScheme
	// Optional: Custom headers to set in the request. HTTP allows repeated headers.
	// +optional
	HTTPHeaders []HTTPHeader
	// ReasonHeader is the header that will be set to the reason for a 
	// deletion. This header defaults to "Kube-Delete-Reason"
	ReasonHeader string
}
```

PreStopHandler is introduced to specialize the current Handler implementation to provide a termination reason during the execution of Lifecycle's PreStop handler.
```
// PreStopHandler invokes either a DeleteExecAction or a DeleteHTTPGetAction 
// prior to the graceful termination of a Pod.
type   struct {
   	// One and only one of the following should be specified.
   	// Exec specifies the action to take.
   	// +optional
   	Exec    *DeleteExecAction
   	// HTTPGet specifies the http request to perform.
   	// +optional
   	HTTPGet *DeleteHTTPGetAction
}
```

The Lifecycle struct is modified such that PreStop is a PreStopHandler.
```
type Lifecycle struct {
    // Other fields and existing comments omitted for brevity.
 
    PreStop *PreStopHandler
}
```

#### API Server
The API Server is responsible for validation and reason propagation from the initiating request to the ObjectMeta corresponding to the request's subject.
In addition to the existing validation performed for the Lifecycle struct, the API Server should fail validation if PreStop is not nil, and if one of the following are true.

1. Both the PreStop's Exec and HTTPGet are nil.
2. Both the PreStop's Exec and HTTPGet are not nil.

Validation for DeleteExecAction and DeleteHTTPGetAction is analogous to the validation performed for ExecAction and HTTPGetAction with the following exceptions.

1. A DeleteExecAction's ReasonEnv must be a valid Linux environment variable name.
2. A DeleteHTTPGetAction's ReasonHeader must be a valid HTTP header name.

When the API Server performs its graceful delete processing, in addition to setting the DeletionTimestamp of the subject ObjectMeta, if the DeleteOptions contains a Reason, the Reason should be copied to the DeletionReason field of the ObjectMeta.

#### Kubelet
Kubelet will supply a termination reason via a configured pre-stop lifecycle hook under the following conditions.

1. The Pod has been explicitly deleted with a supplied DeletionReason.
2. The Pod is evicted due to memory pressure, disk pressure, or an intolerable taint.

In both cases Kubelet will deliver a termination reason to containers with a declared PreStop handler. If the termination is due to an eviction, as described above, Kubelet will supply a termination reason of "Eviction".

##### Termination Reason Delivery
When Kubelet processes the PreStop Handlers of a Pod's containers, prior to sending a TERM signal and starting the termination grace timer, if a termination reason is to be delivered, Kubelet will do the following.

1. If the PreStop handler indicates a DeleteExecAction, Kubelet will set the configured environment variable to the value of the reason prior to execution of the handler.
2. If the PreStop handler indicates a DeleteHTTPGetAction, Kubelet will supply the termination reason to the specified endpoint by setting the configured header to the value of the reason prior to execution of the handler.

#### Kubectl
Kubectl will use the --reason parameter to allow users to pass a arbitrary string as the termination reason as shown below. This parameter is applicable to both the delete and drain verbs.

```
> kubectl delete po my-pod --reason="resolve issue 354961"
```

Kubectl will simply populate the Reason field of the DeleteOptions for the relevant request with the supplied reason.


### Risks and Mitigations

TBD

## Graduation Criteria

Approval by community.
Testing with a real-world workload (eg. cassandra)

## Implementation History

Proposal stage

## Drawbacks

The proposal targets parameterizing an existing lifecycle hook with a reason. This is an incremental improvement vs looking at lifecycle hooks more broadly.

## Alternatives

No alternatives exist today for a pod to modify its termination behaviour.
Alternative for watchers is to observe system logs and events to deduce the reason.
