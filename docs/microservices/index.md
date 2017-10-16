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

These benefits don't come for free. Some of the challenges when adopting a microservices architecture include:

- Design. What are the service boundaries? How big should a service be? How will services communicate?
- Team structure. Can you successfully organize into small, semi-independent teams? Do you have a strong DevOps culture? 
- CI/CD. You must have automated and robust CI/CD, so that you can reliably deploy the application into test and production environments.
- Testing. The decentralized nature of building microservices requires new approaches to testing. During development, you will test an individual service against stubs or API contracts. Then perform integration testing in a production-like test environment (test cluster). Performance tests and load tests are also critical.
- Monitoring and telemetry. A single client request may invoke multiple services. That makes it harder to get a holistic view of the system and understand the sources of bottlenecks and performance issues.

## The drone delivery scenario

## Choosing a compute option







