# Implement an API gateway for microservices

An API gateway sits between clients and services, routing requests from clients to the right services. It may also perform various cross-cutting tasks such as monitoring, authentication, and SSL termination. This section describes some of reasons to use an API gateway, and considerations for implementing one.

![](./images/gateway.png)

## What is an API gateway?

In a microservices architecture, how should a client communicate with the various services that compose the application? One option is for services to expose public endpoints and have clients make direct HTTP calls to them. There are potential problems with this approach, however. 

- A single operation might require calls to multiple services. That can result in multiple network round trips between the client and the server, adding significant latency. 
- It can result in complex client code. The client must keep track of multiple endpoints, and handle failures in a resilient way. 
- It creates coupling between the client and the backend. The client needs to know how the individual services are decomposed. That makes it harder to maintain the client and also harder to refactor services.
- Each individual service must handle concerns such as authentication, SSL, and client rate limiting. 
- Services must expose a client-friendly protocol such as HTTP or WebSocket. This limits the choice of [communication protocols](./interservice-communication.md) 

To address these problems, an API gateway sits between the clients and the backend services. A gateway can perform a number of different functions. Depending on your scenario, you may not need all of them. These functions can be grouped into the following patterns:

[Gateway Routing](../patterns/gateway-routing.md). Use the gateway as a reverse proxy to route requests to one or more backend services, using layer 7 routing. The gateway provides a single endpoint for clients, and helps to decouple clients from services. 

[Gateway Aggregation](../patterns/gateway-aggregation.md). Use the gateway to aggregate multiple individual requests into a single request. This pattern applies when a single operation requires calls to multiple backend services. The client sends one request to the gateway. The gateway dispatches requests to the various backend services, and then aggregates the results and sends them back to the client. This helps to reduce chattiness between the client and the backend. 

[Gateway Offloading](../patterns/gateway-offloading.md). Use the gateway to offload functionality from individual services to the gateway, particularly cross-cutting concerns. It can be useful to consolidate these functions into one place, rather than making every service responsible for implementing them. This is paricularly true for features that requires specialized skills to implement correctly, such as authentication and authorization. 

Here are some examples of functionality that could be offloaded to a gateway:

- SSL termination
- Authentication
- IP whitelisting
- Client rate limiting (throttling)
- Logging and monitoring
- Response caching
- Web application firewall

## Choosing a gateway technology

Some of the options for implementing gateway functionality include:

- [Azure Application Gateway](/azure/application-gateway/). Application Gateway is a managed load balancing service that can perform layer-7 routing and SSL termination. It also provides a web application firewall (WAF).

- [Azure API Management](/azure/api-management/). API Management is a turnkey solution for publishing APIs to external and internal customers. It provides features that are useful for managing a public-facing API, including rate limiting, IP white listing, and authentication using Azure Active Directory or other identity providers. 

- Reverse proxy server, such as Nginx or HAProxy. Nginx and HAProxy are popular reverse proxy servers that support features such as load balancing, SSL, and layer 7 routing. They are both free, open-source products, with paid editions that provide additional features and support options. Nginx and HAProxy are both mature products with rich feature sets and high performance. You can extend them with third-party modules or by writing custom scripts in Lua. Nginx also supports a JavaScript-based scripting module called NginScript.

- Service mesh. If you are using a service mesh such as linkerd or Istio, consider the features that are provided by the ingress controller for that service mesh. For example, the Istio ingress controller supports layer 7 routing, HTTP redirects, retries, and other features.

When choosing a gateway technology, consider the following:

**Features**. The options listed above all support layer 7 routing, but support for other features will vary. 

**Deployment**. Azure Application Gateway and API Management are managed services. Nginx and HAProxy will typically run in containers inside the cluster, but can also be deployed to dedicated VMs outside of the cluster. 

**Management**. When services are updated or new services are added, the gateway routing rules may need to be updated. Consider how this process will be managed. Similarly considerations apply to managing SSL certificates, IP whitelists, and other aspects of configuration.

## Deployment considerations

### Deploying Nginx or HAProxy to Kubernetes

You can deploy Nginx or HAProxy to Kubernetes by creating a ReplicaSet or DaemonSet that specifies the Nginx or HAProxy container image. 

- To achieve high availability, set the replica count higher than 1 for redundacy. The gateway is a potential bottleneck or single point of failure in the system, so it's crucial that the gateway be scalable and high available.

- Create a service of type LoadBalancer to expose the gateway through an Azure Load Balancer

- Use a ConfigMap to store the configuration file for the proxy, and mount the ConfigMap as a volume. 

- Configure a readiness probe that serves a static file from the gateway (rather than routing to another service).

Another alternative is to use an **Ingress Controller**. This lets you configure the reverse proxy as a Kubernetes resource. An Ingress Controller actually involves two separate Kubernetes resources, the *Ingress* and the *Ingress Controller*.

- The Ingress is a resource that defines the configuration for the reverse proxy, such as routing rules, TLS certificates, and...

- The Ingress Controller performs the reverse proxying. Several implementations exist, including Nginx and HAProxy. The Ingress Controller watches the Ingress resource and updates the reverse proxy as needed.

An advantage of using an Ingress Controller is that it abstracts the ingress rules from the implementation details of the reverse proxy. You don't need to manage configuration files or container images. Ingress Controllers are still a beta feature of Kubernetes at the time of this writing, and the feature will continue to evolve.

Consider running the gateway on a dedicated set of nodes in the cluster. Benefits to this approach include:

- Isolation. All inbound traffic goes to a fixed set of nodes, which can be isolated from backend services.

- Stable configuration. If the gateway is misconfigured, the entire application may become unavailable. 

- Performance. You may want to use a specific VM configuration for the gateway for performance reasons.

- Load balancing. You can configure the external load balancer so that requests always go to a gateway node. That can save a network hop, which would otherwise happen whenever a request lands on a node that isn't running a gateway pod. This consideration applies mainly to large clusters, where the gateway runs on a relatively small fraction of the total nodes.   

In Azure Container Service (ACS), this approach currently requires [ACS Engine](https://github.com/Azure/acs-engine)) which allows you to create multiple agent pools.

### Azure Application Gateway

To connect Application Gateway to a Kubernetes cluster in Azure:

1. Create an empty subnet in the cluster VNet.
2. Deploy Application Gateway.
3. Create a Kubernetes service with type=[NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport). This exposes the service on each node so that it can be reached frmo outside the cluster. It does not create a load balancer.
5. Get the assigned port number for the service.
6. Add an Application Gateway rule where:
    - The backend pool contains the agent VMs.
    - The HTTP setting specifies the service port number.
    - The gateway listener listens on ports 80/443
    
Set the instance count to 2 or more for high availability.

### Azure API Management 

To connect API Management to a Kubernetes cluster in Azure:

1. Create an empty subnet in the cluster VNet.
2. Deploy API Management to that subnet.
3. Create a Kubernetes service of type LoadBalancer. Use the [internal load balancer](https://kubernetes.io/docs/concepts/services-networking/service/#internal-load-balancer) annotation to create an internal load balancer, instead of an Internet-facing load balancer, which is the default.
4. Find the private IP of the internal load balancer, using kubectl or the Azure CLI.
5. Use API Management to create an API that directs to the private IP address of the load balancer.

Consider combining API Management with a reverse proxy, whether Nginx, HAProxy, or Azure Application Gateway. 

> [!NOTE]
> To use API Management with Application Gateway, see [Integrate API Management in an internal VNET with Application Gateway](/azure/api-management/api-management-howto-integrate-internal-vnet-appgateway)


