---
id: 0012
title: Event driven design for Tinkerbell
status: ideation
authors: Gaurav Gahlot <gauravgahlot0107@gmail.com>, Gianluca Arbezzano <gianarb92@gmail.com>
---

## Summary

In the current model, the whole tinkerbell stack works on a _request-response_ model.
Instead of waiting for explicit requests or responses, we want the components to watch for certain events and act accordingly.
This introduces extensibility and allows users to plug-and- play with tink.

## Goals and no-Goals

Goal:

- Move Tinkerbell in the event-driven design direction.
- Start small and have a pluggable base ready.
- Be able to handle the `phone-home` with the new event-driven system.

No-Goal:

- Disrupt the current working environment.
- Update workers to receive actions via event driven model.

## Content

Not everything can be a Tink responsibility.
Events are a scalable way to build an extendible system.
This allows different components to tap-in to the event streams and leverage the extension points.
For example, [tinkerbell/portal](https://github.com/tinkerbell/portal/) can watch for workflow events and present them on the UI.
Events are good for troubleshooting purpose because they help to build context.
Feature requests we received that can be implemented using events are: phone home, ontimeout and so on

### Current Model

In the current model, the tink-server is loaded with tons of responsibilities.
Responsibilities like - managing workflow state, handing overs actions to workers, logging the events, and others.
We would like to delegate these responsibilities to respective smaller services/components.
Instead of waiting for an explicit request, these components will be watching for certain events and act accordingly.

We have a client for each resource and it has a bunch of commands (CRUD).

```
// client for Template resource
type TemplateClient interface {
	CreateTemplate(ctx context.Context, in *WorkflowTemplate, opts ...grpc.CallOption) (*CreateResponse, error)
	GetTemplate(ctx context.Context, in *GetRequest, opts ...grpc.CallOption) (*WorkflowTemplate, error)
	DeleteTemplate(ctx context.Context, in *GetRequest, opts ...grpc.CallOption) (*Empty, error)
	ListTemplates(ctx context.Context, in *Empty, opts ...grpc.CallOption) (Template_ListTemplatesClient, error)
	UpdateTemplate(ctx context.Context, in *WorkflowTemplate, opts ...grpc.CallOption) (*Empty, error)
}

// client for Hardware resource
type HardwareServiceClient interface {
	Push(ctx context.Context, in *PushRequest, opts ...grpc.CallOption) (*Empty, error)
	ByMAC(ctx context.Context, in *GetRequest, opts ...grpc.CallOption) (*Hardware, error)
	ByIP(ctx context.Context, in *GetRequest, opts ...grpc.CallOption) (*Hardware, error)
	ByID(ctx context.Context, in *GetRequest, opts ...grpc.CallOption) (*Hardware, error)
	All(ctx context.Context, in *Empty, opts ...grpc.CallOption) (HardwareService_AllClient, error)
	Watch(ctx context.Context, in *GetRequest, opts ...grpc.CallOption) (HardwareService_WatchClient, error)
	Delete(ctx context.Context, in *DeleteRequest, opts ...grpc.CallOption) (*Empty, error)
}

// client for Workflow resource
type WorkflowSvcClient interface {
	CreateWorkflow(ctx context.Context, in *CreateRequest, opts ...grpc.CallOption) (*CreateResponse, error)
	GetWorkflow(ctx context.Context, in *GetRequest, opts ...grpc.CallOption) (*Workflow, error)
	DeleteWorkflow(ctx context.Context, in *GetRequest, opts ...grpc.CallOption) (*Empty, error)
	ListWorkflows(ctx context.Context, in *Empty, opts ...grpc.CallOption) (WorkflowSvc_ListWorkflowsClient, error)
	GetWorkflowContext(ctx context.Context, in *GetRequest, opts ...grpc.CallOption) (*WorkflowContext, error)
	ShowWorkflowEvents(ctx context.Context, in *GetRequest, opts ...grpc.CallOption) (WorkflowSvc_ShowWorkflowEventsClient, error)
	GetWorkflowContextList(ctx context.Context, in *WorkflowContextRequest, opts ...grpc.CallOption) (*WorkflowContextList, error)
	GetWorkflowContexts(ctx context.Context, in *WorkflowContextRequest, opts ...grpc.CallOption) (WorkflowSvc_GetWorkflowContextsClient, error)
	GetWorkflowActions(ctx context.Context, in *WorkflowActionsRequest, opts ...grpc.CallOption) (*WorkflowActionList, error)
	ReportActionStatus(ctx context.Context, in *WorkflowActionStatus, opts ...grpc.CallOption) (*Empty, error)
	GetWorkflowData(ctx context.Context, in *GetWorkflowDataRequest, opts ...grpc.CallOption) (*GetWorkflowDataResponse, error)
	GetWorkflowMetadata(ctx context.Context, in *GetWorkflowDataRequest, opts ...grpc.CallOption) (*GetWorkflowDataResponse, error)
	GetWorkflowDataVersion(ctx context.Context, in *GetWorkflowDataRequest, opts ...grpc.CallOption) (*GetWorkflowDataResponse, error)
	UpdateWorkflowData(ctx context.Context, in *UpdateWorkflowDataRequest, opts ...grpc.CallOption) (*Empty, error)
}
```

### The New Model

The new model is inspired from Kubernetes.
The resource clients should have a `Watch` function that will stream events for that particular resource.

```
Watch(ctx context.Context, in EventWatchRequest)
```

## The API

### Data Model

ResourceType - a resource that an event can be associated with

- Template
- Hardware
- Workflow

EventType - an event type in tinkerbell space(a non-exhaustive list)

- CREATED
- UPDATED
- DELETED
- WORKFLOW_STARTED
- WORKFLOW_INPROGRESS
- WORKFLOW_FAILED
- WORKFLOW_TIMEOUT

Event - an event in tinkerbell space; and has the following structure:

```
{
  "id": "uuid",
  "resourceID": "uuid",
  "resourceType": "workflow",
  "eventType": "created",
  "time": "",
  "data": {
  }
}
```

- `id`: a unique identifier (UUID) for the event occured in tinkerbell space
- `resourceID`: a unique identifier (UUID) for the resource, the event is associated with.
  For example, `resourceID` will be set to `workflowID` for workflow resource and to `templateID` for the template resource.
- `resouceType`: type of the resource (template, hardware, workflow) for which the event was generated
- `eventType`: the event verb; describing the action on resource that generated the event
- `time`: the event timestamp
- `data`: the primary event payload, represented as `interface{}`.
  For a workflow created event, the payload (data) can be the complete workflow structure.
  For an on-timeout event the data can be an action that needs to be executed next.

### EventClient

A new `EventClient` should be developed with the primitive required for the events at least `Watch`.
At the beginning, only a `Watch` function is required but we think a natural evolution will be to serve a function that can be used from other components to fire events.
Or maybe we can do a `Watch` and `Create` straight away.
All the resource clients will have a `Watch` function that will stream events for that particular resource.

```
Watch(ctx context.Context, in EventWatchRequest)
```

Where the `EventWatchRequest` can take the following structure:

```
type EventWatchRequest {
    EventName  // the name of the event to filter by
    ResourceID // the resource ID (ResourceType required when this is set)
    ResourceType // workflow, template, action, hardware
}
```

So when we create a `HardwareClient`, for instance, it would return a watcher for the `Hardware` resource type.
All the clients like `HardwareClient` will be using `EventClient` under the hood.

There is no logic to keep track of which events are fired or sent to a consumer.
Every consumer when it connects to a stream of events will specify how old the events returned should be (by default 5m).

The tink-server will only be responsible for generating the events.
It will not contain any business logic that needs to be executed as an event occurs.
Instead it's the consumer who will have to implement the business logic in an idempotent way.
Forcing the consumer to implement the logic in a way that can be run repeatedly simplifies the logic in Tink but it also enforce reliability in the client side.
If a client fails half way the client gets the same events again and repeat the logic.

```
informer := client.WorkflowClient.Watch(&request.WatchRequest {
			EventType: "WORKFLOW_TIMEOUT"
		}, 
		func(e Event) {
			// Do your best. You can notify via Slack. Or start a different workflow.
		})
informer.Run(ctx)
```

The `Watch` gRPC function needs to support filtering by a `ResourceType` and/or a `ResourceID` and/or an `EventType`.
In case of the `Watch` is called by a resource client, for example HardwareClient, the `ResourceType` filter is fixed.

## System-context-diagram

![system-architecture](architecture.png)

## Refrences

- https://gianarb.it/blog/kubernetes-shared-informer
- https://www.infoq.com/podcasts/kubernetes-event-driven-architecture/
- [Martin Fowler’s talk](https://www.youtube.com/watch?v=STKCRSUsyP0)

## Alternatives

The alternative are:

- We can tight Tinkerbell to a streaming platform or a queuing system like Kafka, RabbitMQ
- We can build a queue in Tinkerbell itself
````