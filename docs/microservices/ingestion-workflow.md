# Ingestion and workflow




![](./images/ingestion-workflow.png)

The Delivery Scheduler service is responsible for accepting user requests and executing a workflow. For each request, the workflow consists of several steps, carried out by calling the various backend services:

1. Check the status of the customer's account (Account service).
2. Create a new package entity (Package service).
3. Create a new delivery entity (Delivery service).
4. Check whether any third-party transportation is required for this delivery, based on the pickup and delivery locations (Third-party Transportation service).
5. Schedule a drone for pickup (Drone service).

If any of these services returns an error code or experiences a non-transient failure, the delivery cannot be scheduled. An error code might indicate an expected error condition &mdash; for example, the customer's account is suspended mdash; or an unexpected server error (HTTP 5xx). A service might also be unavailable, causing the network call to time out. 

Based on business requirements, Fabrikam has identified the following non-functional requirements for ingestion:

- Sustained throughput of 10K requests/sec.
- Handle spikes of up to 100K/sec without dropping client requests or timing out.
- [Latency?]



The requirement to handle occasional spikes in traffic presents a design challenge. In theory, the system could be scaled out to handle the maximum expected traffic. However, provisioning that many resources woud be very inefficient. Most of the time, the application will not need that much capacity, so there would be idle cores and excess database resources, costing money without adding value.

A better approach is to put the incoming requests into a buffer, and let the buffer act as a load leveler. With this design, the Delivery Scheduler must be able to the maximum ingestion rate over short periods, but the backend services only need to handle the maximum sustained load of 5K. By buffering at the front end, the backend services shouldn't need to handle large spikes in traffic.

## Asynchronous operations over HTTP
 
[talk about 200 Accepted here]

## Ingestion

At the scale Fabrikam is targeting, Event Hubs is a good choice, because of its high ingestion rate. Our tests showed that ingress per event hub was about 32k ops/sec with latency around 90ms. The delivery scheduler is also capable of sharding across more than one event hub. Ingress with 2 event hubs was 45k ops/sec with latency below 100 ms. (As with all performance metrics, there can be multiple factors that affect performance, so don't interpret these numbers as a benchmark.)

Service Bus with premium messaging is fast, but still not as fast as Event Hubs. You can see performance results in this blog post. However, keep in mind that these results do not use Service Bus features such as deduplication, peek lock (which is needed to guarantee at-least-once delivery), or dead letter queuing. Also at this scale, Event Hubs is more cost effective than Service Bus. 

However, it's important to understand that Event Hubs is able to achieve this level of throughput because it has fundamentally different semantics than Service Bus queues. This has consequences for how the workload is implemented.

With a queue, an individual consumer can remove a message from the queue, and the next consumer will get the next message on the queue. You can use a competing consumer pattern to process messages in parallel and improve scalability. For greater resiliency, the consumer holds a lock on the message and releases the lock when it's done processing the message. If the consumer fails &mdash; for example, the node it's running on crashes &mdash; the lock times out and the message goes back onto the queue. 

![](./images/queue-semantics.png)

Event Hubs, on the other hand, uses streaming semantics. Consumers read the stream independently at their own pace. Each consumer must keep track of its current position in the stream, using a client-side cursor. 

For resiliency, the consumer should record its position by writing the latest offset to a persistent store at some predefined interval. This process is called checkpointing. If the consumer fails, another instance can pick up from the last checkpoint. In that case, some events may be replayed, depending on when the last checkpoint was saved. There is a tradeoff: Frequent checkpoints are slower, but sparse checkpoints mean you will replay more events after a failure. If you don't checkpoint, it's possible to lose messages if a node fails. For some scenarios, you may be able to tolerate some message loss. But in the case of the Delivery Scheduler, each message represents a customer trying to use the drone service, so it's important to avoid lost messages.

![](./images/stream-semantics.png)
 
Event Hubs is not designed for competing consumers. Although multiple consumers can read a stream, each traverses the stream independently. Instead, Event Hubs uses a partitioned consumer pattern. An event hub has up to 32 partitions. Horizontal scale is achieved by assigning a separate consumer to each partition.

What does this mean for the drone delivery workflow? To get the full benefit of Event Hubs, the Delivery Scheduler cannot wait for each message to be processed before moving onto the next. If it does that, it will spend most of its time waiting for network calls to complete. Instead, it needs to process batches of messages in parallel, using asynchronous calls to the backend services. As we'll see, choosing the right checkpointing strategy is also important.  


## Workflow

We looked at three options for reading and processing the messages: Event Processor Host, Service Bus queues, and the IoTHub React library. (Spoiler alert: We eventually chose IoTHub React, for reasons that we'll get into.) 

### Event Processor Host

Event Processor Host is designed for message batching. The application implements the `IEventProcessor` interface, and the Processor Host creates one `IEventProcessor` instance for each partition in the event hub. The Processor Host invokes the `IEventProcessor.ProcessEventsAsync` method with a batch of event messages.

Horizontal scaling is achieved by having each Processor Host instance compete to hold a lease on the available partitions. Over time, partition leases are distributed evenly across Processor Host instances. 

![](./images/partition-lease.png)

Event Processor Host writes checkpoints to Azure storage. The application controls when checkpointing occurs by calling `CheckpointAsync` inside the ProcessEventsAsync method.

For each batch of events, the processor host waits until the `ProcessEventsAsync` method returns before calling the method again with the next batch. This means that the processor host behaves in a synchronous manner as far as each instance of IEventProcessor is concerned.

> [AZURE.NOTE] 
> The processor host doesn't actually *wait* in the sense of blocking a thread. The `ProcessEventsAsync` method is asynchronous, so the processor host continues to do other work while the method is completing. But it won't deliver another batch of messages to that instance until the method returns.

This approach simplifies the programming model, because your `IEventProcessor` implementation does not have to be reentrant. However, it also means that the event processor handles one batch at a time, and this gates the speed at which the Processor Host can pump messages.

In our scenario, each request message is independent, so we can process them in parallel. But waiting for an entire batch to complete can still cause a bottleneck,because processing can only be as fast as the slowest message within a batch. Any variation in response times can result in a "long tail" where a few slow responses drag down the entire system. And in fact, our performance tests showed that we did not achieve our target throughput using this approach. 

This does *not* mean that you should avoid using Event Processor Host. The real lesson to avoid performing long-running tasks inside the `ProcesssEventsAsync` method. 

### Service Bus queues

Our next idea was to copy the messages from Event Hubs over to a Service Bus queue, and then read the messages from Service Bus to process them. This idea may seem contradictory, given that we had already decided not to use Service Bus for ingestion. However, the idea was to leverage the different strengths of each service: Use Event Hubs to absorb spikes of heavy traffic, while taking advantage of the queue semantics in Service Bus to process the workload, using a competing consumer model. Remember that our target for sustained throughput is less than our expected peak load.
 
With this approach, our proof-of-concept implementation achieved about 4K operations per second [NEED ACTUAL NUMBERS HERE]. These tests used mock backend services that did not do any real work, but simply added a fixed amount of latency per service. Note that our performance numbers were much less than the theoretical maximum for Service Bus. Possible reasons for the discrepancy include:

- Not having optimal values for various client parameters, such as the connection pool limit, the degree of parallelization, the prefetch count, and the batch size.

- Network I/O bottlenecks.

- Use of PeekLock mode rather than ReceiveAndDelete, which was needed to ensure at-least-once delivery of messages

Further load testing might have discovered the root cause and allowed us to resolve these issues. However, a parallel effort using IotHub React showed promise, so we switched to that approach.

## IotHub React 

[IotHub React](https://github.com/Azure/toketi-iothubreact) is an Akka Streams library for reading events from Event Hub. Akka Streams is a stream-based programming framework that implements the Reactive Streams specification. It provides a way to build efficient streaming pipelines, where all streaming operations are performed asynchronously, and the pipeline gracefully handles backpressure. Back pressure occurs when an event source produces events at a faster rate than the downstream consumers can receive them &mdash; which is exactly the situation when the drone delivery system has a spike in traffic. If backend services go slower, IoTHub React will slow down. If capacity is increased, IoTHub React will push more messages through the pipeline.

Akka Streams is also a very natural programming model for streaming events from Event Hubs. Instead of looping through a batch of events, you define a set of operations on events, and let Akka Streams handle the streaming. 

[Show code here]

IoTHub React uses a different checkpoint strategy than Event Host Processor. The checkpoint logic resides in a sink, which is the terminating stage in a pipeline. The design of Akka Streams allows the pipeline to continue streaming data while the sink is writing the checkpoint. That means the upstream processing stages don't need to wait on the checkpoint. You can configure chcekpoint to occur after a timeout or after a certain number of messages have been processed. 
 
Considerations for scaling:

- To make it easier to scale out, each instance of the dispatcher service is assigned a single

- To scale the dispatcher has to be deployed as a statefulsets in kubernetes. Like Deployments, StatefulSets manage Pods that are based on an identical container spec. However, although their specs are the same, the Pods in a StatefulSet are not interchangeable. Each Pod has a persistent identifier that it maintains across any rescheduling. In the case of dispatcher the identifier for the container is the partition id that is assigned to each pod running the workflow. Each pod will execute the messages for its own partition. Pods can overlap on the VM giving a cost effective way to distribute workload in a high density model. This is easy to manage through image updates and deployment cycles. 
- Customers with higher scale requirements than the number of partitions in event hub, can implement a hashing algorithm in the dispatcher and deploy more than one readers per partition pointing to different storage accounts. The result of this configuration is that multiple readers will pick up messages overlapping among them but will only process the messages that belong to them, according to their hashing algorithm.
- Correct sizing of nodes and the right density of them in VMS is important aspect on the dispatcher deployment. The dispatcher is memory and thread bound, because of the akka framework. 

## Handling failures

- 


