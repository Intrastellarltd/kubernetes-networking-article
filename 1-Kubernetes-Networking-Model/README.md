# Kubernetes Networking

In this article, we will examine the important topic of networking. Kubernetes as an orchestration platform is capable of managing thousands of pods running on diﬀerent machines, this calls for a robust networking model to start with. The following topics will be touched upon:

- Kubernetes Networking Model
- Kubernetes network plugins
- Kubernetes and eBPF
- Kubernetes networking solutions
- Using network policies eﬀectively
- Load balancing options

---

## Kubernetes Networking Model

The Kubernetes networking model is characterized by a flat address space, where all pods within a cluster have direct visibility to one another. Each pod possesses its own unique IP address, negating the necessity for Network Address Translation (NAT). Furthermore, containers residing in the same pod share the pod's IP address and can communicate internally via the localhost interface. While this model may exhibit a specific perspective, its establishment greatly simplifies the experiences of developers and administrators alike. Its particular benefit emerges when transitioning legacy network applications to Kubernetes. In this framework, a pod corresponds to a conventional node, while each container embodies a traditional process.

The following topics will be covered:

- Container to container communication
- Pod to pod communication
- Pod to service communication
- External access
- Lookup and discovery
- DNS in Kubernetes

### Container to Container Communication

In the Kubernetes environment, a running pod is always allocated to a single physical or virtual node. This arrangement ensures that all containers within the pod operate on the same node and can establish communication using various means, including local filesystem access, inter-process communication mechanisms, and utilizing the localhost interface along with familiar ports. The absence of port collisions between different pods is guaranteed due to each pod's distinct IP address. When a container within a pod uses localhost, this pertains exclusively to the pod's IP address. For instance, if container 1 in pod 1 connects to port 1234—a port listened to by container 2 in the same pod—it won't conflict with another container in a different pod on the same node that also employs port 1234. However, when exposing ports to the host, it's essential to exercise caution regarding pod-to-node affinity. This challenge can be effectively managed through mechanisms like Daemonsets and pod anti-affinity.

### Pod to Pod Communication

In Kubernetes, pods are assigned IP addresses that are visible on the network (not confined to the node), enabling direct communication without the need for NAT, tunnels, or proxies. This setup allows the use of well-known port numbers for straightforward communication. The internal IP address of a pod aligns with its external IP address as perceived by other pods within the cluster network, although it remains concealed from the external world. Consequently, standard naming and discovery mechanisms like Domain Name System (DNS) operate seamlessly without additional configuration.

### Pod to Service Communication

Pods within a Kubernetes cluster can communicate directly via IP addresses and well-known ports. However, given the dynamic nature of pod creation and destruction, knowing each pod's IP address becomes impractical. To address this, Kubernetes employs the service resource as a stable and indirection layer. This layer remains consistent even as the set of actual pods responding to requests evolves.

![Internal load balancing using a serviceExternal access](/architecture-diagram/Internal%20load%20balancing%20using%20a%20serviceExternal%20access.png)

Additionally, when some containers need to be accessible externally, the pod IP addresses remain concealed. External access is achieved through services, but often requires two redirects. For instance, cloud provider load balancers lack Kubernetes awareness and direct traffic to a cluster node rather than to a specific pod-handling request. Subsequently, the kube-proxy on the node redirects the traffic to the appropriate pod if the current node isn't hosting the required pod.

The diagram illustrates how an external load balancer sends traffic to an arbitrary node, and then the kube-proxy manages further routing if necessary.

![External load balancer sending traffic to an arbitrary node and the kube-proxy](/architecture-diagram/External%20load%20balancer%20sending%20traffic.png)

### Lookup and Discovery

For effective communication between pods and containers, it's crucial for them to discover each other's presence. There exist multiple methods by which containers can locate other containers or announce their own availability.

#### Self-Registration

Containers within a pod can communicate effectively by registering their IP addresses and ports with a dedicated registration service. This service allows containers to find each other and establish connections. When a container launches, it can self-register by connecting to the registration service and sharing its IP address and port. Other containers can then query the registration service to retrieve the IP addresses and ports of all registered containers, enabling direct connections.

Self-registration offers benefits such as simplified container tracking and the ability for containers to apply intelligent policies. Containers can temporarily unregister themselves based on local conditions, enhancing dynamic load balancing. However, ungraceful container termination requires mechanisms like periodic pinging or keep-alive messages to detect unavailable containers. While this approach ensures efficient communication and decentralized load balancing, it introduces the need for an additional component, the registration service, which containers must be aware of to locate and interact with other containers.

#### Services and Endpoints

Kubernetes services function as standardized registration services. Pods associated with a service are automatically registered through their labels. Other pods can access service endpoints to discover all the pods linked to the service or communicate directly with the service, which routes the message to a backend pod.

Typically, pods send messages to the service itself, which then forwards them to one of the backend pods. To maintain dynamic membership, a combination of factors like deployment replica counts, health checks, readiness checks, and horizontal pod autoscaling can be utilized.

#### Loosely Coupled Connectivity with Queues

Containers can communicate seamlessly without requiring knowledge of IP addresses, ports, or service identifiers by utilizing asynchronous and decoupled communication. This approach is particularly useful for systems comprised of loosely coupled components that remain unaware of one another's existence. Queues serve as facilitators for such loosely coupled systems. Containers listen to messages from queues, respond, perform tasks, and post messages back to the queue for tracking purposes.

Queues offer various advantages:

- Scalability is easily achieved by adding more containers that listen to the queue, requiring no coordination.
- Overall load can be gauged based on queue depth.
- Multiple component versions can coexist by versioning messages or queue topics.
- Load balancing and redundancy can be implemented by having multiple consumers processing requests in different modes.
- Dynamic addition or removal of various listeners is straightforward.

However, there are downsides to using queues:

- Ensuring queue durability and high availability is crucial to prevent it from becoming a single point of failure (SPOF).
- Containers must interact with the asynchronous queue API, which may need abstraction.
- Implementing request-response interactions can be somewhat complex when using response queues.

In summary, queues offer an excellent mechanism for large-scale systems, providing coordination benefits even within extensive Kubernetes clusters.

#### Kubernetes Ingress

Kubernetes provides an ingress resource and controller that facilitate the exposure of Kubernetes services to external entities. While it's possible to handle this process manually, many tasks involved in defining an ingress are typically shared among applications of a similar type, such as web applications, CDNs, or DDoS protectors. It's also an option to craft custom ingress objects.

The ingress object is frequently employed for intelligent load balancing and TLS termination. Instead of setting up and deploying a separate Nginx server, users can leverage the inherent capabilities of the built-in ingress controller.

### DNS in Kubernetes

DNS (Domain Name System) plays a pivotal role in networking by offering a hierarchical and decentralized naming system that introduces an abstraction layer over IP addresses. This technology is essential for various use cases, including load balancing, dynamic host replacement with varying IP addresses, and assigning human-friendly names to recognizable access points.

In Kubernetes, the core addressable entities are pods and services. Every pod and service possesses a unique internal IP address within the cluster. Pods are equipped with a resolve.conf file by the kubelet, directing them to the internal DNS server.

By default, a pod's hostname corresponds to its metadata name. However, for pods to possess fully qualified domain names within the cluster, the creation of a headless service is necessary. Additionally, the hostname can be explicitly defined along with a subdomain linked to the service name.

#### CoreDNS

The internal DNS server in Kubernetes, initially named kube-dns, resides within the kube-system namespace. Over time, CoreDNS took over as the primary DNS server in Kubernetes due to its improved performance and simplified architecture, replacing the previous kube-dns default.
