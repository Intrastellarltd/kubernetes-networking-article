# Kubernetes Networking

---

## Load balancing options

Load balancing is a vital aspect of dynamic systems like Kubernetes, where nodes, virtual machines (VMs), and pods frequently enter and exit the environment. Clients connecting to the cluster face challenges in keeping track of available entities to handle their requests. Client-side load balancing, where clients manage this complexity, can be complex and inefficient. Instead, server-side load balancing is a proven method that abstracts this complexity away from clients.

There are various load balancing options available, both for external and internal traffic:

1. **External Load Balancer**: Directs incoming external traffic to different nodes or pods within the cluster.

2. **Service Load Balancer**: Kubernetes services provide built-in load balancing for internal traffic among pods.

3. **Ingress**: Acts as an external entry point for HTTP and HTTPS traffic, managing routing rules and load balancing.

4. **HA Proxy**: A popular open-source software load balancer that can be integrated with Kubernetes.

5. **MetalLB**: An external load balancer implementation for bare-metal clusters.

6. **Traefik**: A cloud-native, dynamic reverse proxy and load balancer that integrates with Kubernetes.

7. **Kubernetes Gateway API**: An evolving solution for handling traffic ingress into Kubernetes clusters.

### External load balancers

An external load balancer operates outside the Kubernetes cluster and requires an external load balancer provider. Kubernetes communicates with this provider to set up configurations, such as health checks and firewall rules, and to obtain the external IP address for the load balancer. This setup enables the external load balancer to manage traffic to the cluster's services.

![AWS Load Balancer](https://d2908q01vomqb2.cloudfront.net/fe2ef495a1152561572949784c16bf23abb28057/2022/11/22/pic201.png)

![AWS Load Balancer](https://d2908q01vomqb2.cloudfront.net/fe2ef495a1152561572949784c16bf23abb28057/2022/11/18/Picture2-7.png)

### Understanding external load balancing

External load balancers distribute traffic at the node level. For example, if a service has four pods, with three on node A and one on node B, an external load balancer would divide the load evenly between the two nodes. As a result, the three pods on node A would each handle a third of the load (1/6 each), while the single pod on node B would handle the remaining half of the load. This distribution might be uneven. To address this, weights could be added in the future. To avoid such issues, strategies like pod anti-affinity or topology spread constraints can be used to evenly distribute pods across nodes.

#### Service load balancers

Service load balancing in Kubernetes is intended for internal traffic within the cluster and not for external load balancing. It is achieved using a service type called `clusterIP`. While it's possible to expose a service load balancer externally using the `NodePort` service type, this method has limitations. Curating Node ports to prevent conflicts across the cluster can be challenging and may not be suitable for production environments. Additionally, advanced features like SSL termination and HTTP caching might not be easily accessible with this approach.

#### Ingress

In Kubernetes, an Ingress is a set of rules that enable incoming HTTP/S traffic to reach services within the cluster. In addition to basic routing, certain ingress controllers offer additional features like connection algorithms, request limits, URL rewrites, load balancing for TCP/UDP, SSL termination, and access control.

Ingress is defined using an Ingress resource and is handled by an ingress controller. Kubernetes has two official ingress controllers in its main repository. One is an L7 ingress controller for Google Cloud Engine (GCE) only, while the other is a versatile Nginx ingress controller that allows Nginx configuration via a ConfigMap. The Nginx controller is sophisticated and brings advanced features that may not be available directly through the Ingress resource. It supports multiple platforms like Minikube, GCE, AWS, Azure, and bare-metal clusters.

#### HAProxy

For implementing a custom external load balancer in Kubernetes, there are multiple options available:

1. **LoadBalancer Service Type:** Create a custom external load balancer provider and use the `LoadBalancer` service type to expose your services externally.

2. **NodePort Service Type:** Utilize the `NodePort` service type and carefully manage port allocations to route traffic through nodes for load balancing.

3. **Custom Load Balancer Provider Interface:** Implement a custom solution that interfaces with your external load balancer infrastructure to achieve external load balancing.

4. **HAProxy:** High-Availability (HA) Proxy is a mature and battle-tested load balancing solution that's often considered an excellent choice for implementing external load balancing, especially for on-premises clusters. You can use HAProxy in different ways:

  a. Utilize `NodePort` with HAProxy and manage port allocations.
  
  b. Implement a custom load balancer provider interface tailored to HAProxy.
  
  c. Run HAProxy inside your Kubernetes cluster, using it as the target for your frontend servers at the cluster's edge, whether load balanced or not.

Irrespective of the approach chosen, it's advisable to employ Kubernetes ingress objects for managing and exposing services externally. Notably, the community-driven `service-loadbalancer` project has introduced a load balancing solution built on top of HAProxy.

##### Utilizing the NodePort

In scenarios where a custom external load balancer like HAProxy is employed, the following approach is commonly used:

1. **Port Allocation:** Each Kubernetes service is assigned a dedicated port from a predefined high range (e.g., 30,000 and above) to prevent clashes with other applications using well-known ports.

2. **HAProxy Configuration:** HAProxy operates outside the Kubernetes cluster and is configured with the appropriate port for each service. It acts as the entry point for external traffic.

3. **Double Load Balancing:** HAProxy forwards incoming traffic to any Kubernetes node, which then routes it to the internal service. This introduces an additional hop in the process.

4. **Optimization:** To improve this approach, it's recommended to query the Kubernetes Endpoints API dynamically. By managing the list of backend pods for each service and directly forwarding traffic to these pods, the double load balancing hop can be bypassed.

By adopting this method, external traffic can be efficiently routed to the appropriate pods without the unnecessary additional hop, enhancing performance and reducing latency.

##### HAProxy inside the Kubernetes cluster

HAProxy has developed its own Kubernetes-aware ingress controller, which provides a seamless way to incorporate HAProxy into your Kubernetes environment. By using the HAProxy ingress controller, you can benefit from various capabilities:

1. **Integration:** The controller offers smooth integration with the HAProxy load balancer, ensuring efficient routing of traffic within your Kubernetes cluster.

2. **SSL Termination:** It supports SSL termination, allowing secure communication between clients and the services in your cluster.

3. **Rate Limiting:** The controller enables rate limiting, helping you manage the flow of incoming requests to prevent overload.

4. **IP Whitelisting:** You can enforce IP whitelisting rules to control access to your services.

5. **Load Balancing Algorithms:** Multiple load balancing algorithms are available, including round-robin, least connections, URL hash, and random, providing flexibility in traffic distribution.

6. **Dashboard:** A dashboard provides insights into the health of your pods, request rates, response times, and other relevant metrics.

7. **Traffic Overload Protection:** The controller includes mechanisms to protect against traffic overload, ensuring stable and reliable service performance.

By leveraging the capabilities of the HAProxy ingress controller, you can effectively manage and optimize traffic within your Kubernetes cluster while benefiting from HAProxy's feature-rich load balancing and traffic management capabilities.

##### MetalLB

MetalLB is a load balancer solution specifically designed for bare-metal clusters. It offers high configurability and supports various operational modes, including Layer 2 (L2) and Border Gateway Protocol (BGP).

##### Traeﬁk

Traefik is a modern HTTP reverse proxy and load balancer designed to support microservices. It works with various backends, including Kubernetes, and automatically manages its configuration dynamically. Notable features include:

- Speed and efficiency: Traefik is fast and efficient.
- Single executable: It comes as a single executable, making it easy to deploy.
- Lightweight Docker image: The official Docker image is resource-efficient.
- REST API: Offers a RESTful API for integration and interaction.
- Dynamic configuration reload: Configuration changes can be applied on-the-fly without restarts.
- Circuit breakers and retries: Handles network failures with circuit breakers and retry mechanisms.
- Load balancing algorithms: Supports round-robin and rebalancer algorithms.
- Metrics support: Provides options for metrics collection, including Prometheus, Datadog, etc.
- User-friendly web UI: Features an AngularJS-powered web UI for easy configuration and monitoring.
- Protocol support: Handles WebSockets, HTTP/2, and GRPC protocols.
- Access logs: Provides access logs in JSON and Common Log Format.
- Let's Encrypt integration: Seamlessly works with Let's Encrypt for automatic HTTPS certificate management.
- High availability: Supports cluster mode for redundancy and fault tolerance.

In summary, Traefik offers a robust and feature-rich solution for deploying and managing applications with scalability and reliability in mind.

### Kubernetes Gateway API

The Kubernetes Gateway API is a collection of resources designed to model service networking within Kubernetes. It can be seen as an advancement from the ingress API. While the ingress API will still remain, its limitations couldn't be effectively addressed through enhancements, leading to the creation of the Gateway API project.

Unlike the ingress API, which primarily involves an Ingress resource and an optional IngressClass, the Gateway API takes a more fine-grained approach. It divides the definition of traffic management and routing into distinct resources. The Gateway API introduces the following resources:

- GatewayClass: Defines classes of gateways with specific characteristics.
- Gateway: Represents a gateway instance that handles incoming traffic.
- HTTPRoute: Specifies routing for HTTP traffic.
- TLSRoute: Manages routing for encrypted TLS traffic.
- TCPRoute: Configures routing for TCP traffic.
- UDPRoute: Controls routing for UDP traffic.

Overall, the Gateway API offers a more modular and flexible approach to managing networking and routing configurations in Kubernetes.

#### Gateway API resources

The GatewayClass in the Kubernetes Gateway API establishes standardized settings and behavior that can be shared across multiple gateways.

The primary function of a gateway is to define an entry point into the cluster, along with a set of routes that direct incoming traffic to backend services. Ultimately, the gateway configuration is responsible for configuring the underlying load balancer or proxy to manage this traffic.

Routes play a crucial role in the Gateway API by mapping specific incoming requests that match a defined route to corresponding backend services.

![AWS Load Balancer](https://gateway-api.sigs.k8s.io/images/api-model.png)

#### Attaching routes to gateways

Gateways and routes within the Kubernetes Gateway API can be linked together in various arrangements:

1. One-to-one: A single gateway can be connected to a single route that belongs to a specific owner. This route isn't connected to any other gateways.

2. One-to-many: A gateway has the capability to be connected to multiple routes. These routes may come from different owners.

3. Many-to-many: A route can be associated with multiple gateways, and each of these gateways can have additional routes of their own. This enables flexible connectivity between routes and gateways.
