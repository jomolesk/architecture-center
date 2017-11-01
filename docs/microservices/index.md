# Build and run a microservices architecture on Azure

Microservices have become a popular architectural style for building cloud applications that are resilient, highly scalable, and able to evolve quickly. To be more than just a buzzword, however, microservices requires a different approach to designing and building applications. 

In this set of articles, we explore how to build and run a microservices architecture on Azure. Topics include:

- Using Domain Driven Design (DDD) to design a microservices architecture. 
- Choosing the right Azure technologies for compute, storage, messaging, and other elements of the design.
- Understanding microservices design patterns.
- Analyzing and optimizing performance.
- Building a CI/CD pipeline.

Throughout, we focus on an end-to-end scenario: A drone delivery service that lets customers schedule packages to be picked up and delivered via drone. But first, let's start with fundamentals. What are microservices, and what are the advantages of adopting a microservices architecture?

## Why build microservices?

In a microservices architecture, the application is composed of small, independent services. Here are some of the defining characteristics of microservices:

- Each service implements a single business capability.
- Services run in separate processes, communicating through well-defined APIs or messaging patterns.
- Services do not share data stores or data schemas. Each service is responsible for managing its own data. 
- Services have separate code bases, and do not share source code. They may use common utility libraries, however.
- Services are small. A small team of developers, typically 5 &ndash; 10 people, can write and maintain a service.
- Services are deployed independently. 

Done correctly, microservices can provide a number of useful benefits:

- **Fast release cycles.** Because services are deployed independently, bug fixes and feature releases are more manageable and less risky. You can update a service without redeploying the entire application, and roll back an update if something goes wrong. In many traditional applications, if a bug is found in one part of the application, it can block the entire release process. New features may be held up waiting for a bug fix to be integrated, tested, and published. 

- **Small code, small teams.** A single development team can build, test, and deploy a service. By not sharing code or data stores, dependencies between services are minimized. That makes it easier to add new features. A large code base is harder to understand. There is a tendency over time for code dependencies to become tangled, so that adding a new feature requires touching code in a lot of places. (Have you ever felt that you were spending more time resolving merge conflicts than writing new features?) Having smaller teams can also speed up development. The so-called "two-pizza rule" says that a team should be small enough two pizzas can feed the team (abuot 8 people). Large groups tend be less productive because communication is slower, management overhead goes up, and agility diminishes. Microservices let you scale the two-pizza rule to very large applications. 

- **Mix of technologies**. Teams can pick the technology that best fits their service, using a mix of technology stacks as appropriate. 

- **Resiliency.** A service can go down without taking down the entire application. 

- **Scalability.** At cloud scale, you would like the ability to scale out the parts of the application that are especially resource intensive, without scaling out the entire application. With microservices, services can be scaled independently. At the same time, by running services in containers, you pack a higher density of service instances onto a single VM. 


## No free lunch

These benefits don't come for free. This series of articles is designed to address some of the challenges of building microservices that are resilient, scalable, and manageable.

- **Service boundaries**. When you build microservices, you need to think carefully about where to draw the boundaries between services. Once services are built and deployed in production, it can be hard to refactoring across these boundaries. Yet deciding on these boundaries is one of the big challenges in designing a microservices architecture. How big should a service be? When should functionality be factored into multiple services, or kept in the same service. In this series, we describe an approach that uses domain-driven design to find service boundaries. It starts with [Domain analysis](./domain-analysis.md) to find the bounded contexts, then applies a set of [tactical DDD patterns](./tactical-ddd.md) based on functional and non-functional requirements. 

- **Complexity**. A microservices application has more moving parts. Each service is simple, but the services have to work together as a whole. A single user operation may involve multiple services. In the chapter [Ingestion and workflow](./ingestion-workflow.md), we examine some of the issues around ingesting requests at high throughput, coordinating a workflow, and handling failures. 

- **Network congestion and latency**. The use of many small, granular services can result in more interservice communication and longer end-to-end latency. The chapter [Interservice communication](./interservice-communication.md) describes considerations for messaging between services. Both synchronous and asynchronous communication have a place in microservices architectures. For synchronous communication, good [API design](./api-design.md) is important so that services interoperate cleanly.
 
- **Communciation between clients and the application.**  When you decompose an application into many small services, how should clients communicate with those services? Should a client call each individual service directly, or route requests through an [API Gateway](./gateway.md).

- **Data consistency and integrity**. A basic principle of microservices is that each service manages its own data. This keeps services decoupled, but can lead to challenges with data integrity or redundancy. We explore some of these issues in the [Data considerations](./data-considerations.md).

- **Monitoring**. Monitoring a distributed application can be a lot harder than a monolithic application, because you must correlate telemetry from multiple services. The chapter [Logging and monitoring](./logging-monitoring.md) addresses these concerns.

- **CI/CD**. One of the main goals of microservices is agility. You must have automated and robust [CI/CD](./ci-cd.md), so that you can quickly and reliably deploy individual services into test and production environments. 

- **Team structure**. Can you successfully organize into small, semi-independent teams? Do you have a strong DevOps culture? [Conway's law](https://en.wikipedia.org/wiki/Conway%27s_law) says that organizations create software that mirrors their organizational structure. If your team structure and processes still reflect a "monolithic app" worldview, it will be hard to achieve the agility that microservices promise. Team organization is not a topic that we explore deeply in this series, but it's something to consider before you embark on a microservices architecture.


<!-- - **Testing**. The decentralized nature of building microservices requires new approaches to testing. During development, you will test an individual service against stubs or API contracts. Then perform integration testing in a production-like test environment (test cluster). Performance tests and load tests are also critical.  -->

## The drone delivery scenario

To explore these issues, and to illustrate some of the best practices for a microservices architecture, we created a reference implmentation that we call the Drone Delivery application. 

​Fabrikam, Inc. is starting a drone delivery service. The company manages a fleet of drone aircraft. Businesses register with the service, and users can request a drone to pick up goods for delivery. When a customer schedules a pickup, a backend system assigns a drone and notifies the user with an estimated delivery delivery time. While the delivery is in progress, the customer can track the location of the drone, with a continuously updated ETA.

This scenario involves a fairly complicated domain. Some of the business concerns include scheduling drones, tracking packages, managing user accounts, and storing and analyzing historical data. Moreover, Fabrikam wants to get to market quickly and then iterate quickly, adding new functionality and capabilities. The application needs to operate at cloud scale, with a high SLA. Fabrikam also expects that different parts of the system will have very different requirements for data storage and querying. All of these considerations lead Fabrikam to choose a microservices architecture for the Drone Delivery application.

> [!NOTE]
> For help in choosing between a microservices architecture and other architectural styles, see the [Azure Application Architecture Guide](../guide/index.md).

## Choosing a compute option

The term *compute* refers to the hosting model for the computing resources that your application runs on. The first high-level decision is choosing between an orchestrator or a serverless architecture. With an orchestrator, services run on nodes (VMs) and the orchestrator acts as a control plane. The orchestratory provides functionality such as service discovery, load balancing, configuration, scheduling (placing services on nodes), health monitoring, restarting unhealthy services, network routing, scaling, and so on. With a serverless architecture, you don't manage the VMs or the virtual network infrastructure. Instead, you deploy code and the hosting service handles putting that code onto a VM and executing it. This approach tends to favor small granular functions, that are coordinated by applying event-based triggers to functions. For example, putting a message onto a queue may trigger a function that reads from the queue.

For our Drone Delivery reference implementation, we are using ACS with Kubernetes. Most of the high-level architectural decisions and challenges will apply to any container orchestrator, although specific implementation details will differ. Although we focused on Kubernetes, we did implement one of our microservices as an Azure Function. However, this guide does not focus on serverless architecture in general. 

Options for using an orchestrator in Azure include:

- Azure Container Service (ACS). ACS lets quickly deploy a production ready Kubernetes, DC/OS, or Docker Swarm cluster.
- AKS (Azure Container Service). AKS is a managed Kubernetes service. AKS provisions Kubernetes and exposes the Kubernetes API endpoints, but hosts and manages the Kubernetes control plane, performing automated upgrades, automated patching, autoscaling, and other management tasks. You can think of AKS as being "Kubernetes APIs as a service." At the time of writing, AKS is still in preview. However, it's expected that AKS will become the preferred way to run Kubernetes in Azure.
- Service Fabric. Orchestrates services that can run as containers, binary executables, or as Reliable Services. Reliable Services is a light-weight framework for writing services that integrate with the Service Fabric platform.

To create a serverless architecture on Azure, use Azure Functions combined with Event Grid.
