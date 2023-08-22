# Kubernetes Networking

---

## Kubernetes network plugins

Kubernetes employs a network plugin system to accommodate the diverse range of networking approaches preferred by different users. This flexibility enables support for various scenarios. The central network plugin utilized is CNI (Container Network Interface), which will be examined in-depth. However, Kubernetes also provides a simpler alternative called Kubenet.

Before delving into specifics, it's crucial to grasp the fundamentals of Linux networking, albeit only a fraction of the extensive topic. This knowledge serves as the groundwork for comprehending Kubernetes networking, as it's built upon standard Linux networking principles.

### Basic Linux networking

In Linux, the default setup involves a single network space where all physical network interfaces are accessible. However, this single space can be further divided into multiple logical namespaces. This division is especially significant in the context of container networking.

#### IP addresses and ports

IP addresses serve as identifiers for network entities. Servers have the ability to listen for incoming connections on various ports, while clients can establish connections (TCP) or transmit/receive data (UDP) to servers within their network.

#### Network namespaces

Namespaces gather network devices together, enabling communication among servers within the same namespace while restricting access to servers in other namespaces, even if they're on the same physical network. The connection of networks or network segments is achieved through mechanisms like bridges, switches, gateways, and routing.

#### Subnets, netmasks, and CIDRs

Networks benefit from precise segmentation, particularly when designing and managing them. Dividing networks into smaller subnets with shared prefixes is a common approach. These subnets can be defined using bitmasks that determine the subnet's size, specifying how many hosts it can accommodate. For instance, a netmask of 255.255.255.0 designates the first three octets for routing, leaving room for only 256 (technically 254) individual hosts.

Classless Inter-Domain Routing (CIDR) notation is frequently used due to its conciseness and ability to encode more information. CIDR allows the consolidation of hosts from various legacy classes (A, B, C, D, E). For example, 172.27.15.0/24 indicates that the first 24 bits (3 octets) are allocated for routing purposes.

#### Virtual Ethernet devices

Virtual Ethernet (veth) devices emulate physical network devices. By creating a veth device connected to a physical device, you can allocate this veth (and the corresponding physical device) to a specific namespace. In doing so, devices from other namespaces are unable to directly access it, even if they are physically located on the same local network.

#### Bridges

Bridges link various network segments to form a unified network, facilitating communication between all nodes. This bridging occurs at the data link layer (Layer 2) of the OSI network model.

#### Routing

Routing establishes connections between distinct networks, often using routing tables to guide network devices in directing packets toward their intended destinations. This process involves various network components, including routers, gateways, switches, firewalls, and even standard Linux systems.

#### Maximum transmission unit

The maximum transmission unit (MTU) dictates the maximum packet size, such as 1,500 bytes for Ethernet networks. A larger MTU enhances the payload-to-header ratio, beneficial for efficiency. However, larger MTUs decrease minimum latency as complete packet arrival is required. Additionally, bigger packets demand complete retransmission in case of failures, impacting reliability.

### Kubenet

Kubenet functions as a basic network plugin by setting up a Linux bridge called cbr0 and generating a veth interface for every pod. Cloud providers often utilize this setup to configure routing rules for node-to-node communication, as well as in single-node scenarios. The veth pair links each pod to its host node, employing an IP address from the host's IP address range.

#### Setting the MTU

The Maximum Transmission Unit (MTU) significantly influences network performance. Kubernetes network plugins like Kubenet strive to determine the optimal MTU, but occasional assistance is needed. If an existing network interface, such as the docker0 bridge, employs a smaller MTU, Kubenet adopts it. A similar scenario arises with IPsec, which demands a lower MTU due to added IPsec overhead. However, Kubenet disregards this factor.

To address these complexities, relying on automatic MTU calculation is discouraged. Instead, kubelet should be informed of the desired MTU for network plugins via the "-network-plugin-mtu" command-line switch available to all network plugins. Notably, only the Kubenet network plugin currently integrates this command-line switch.

Primarily present for backward compatibility, the Kubenet network plugin takes a secondary role. The Container Network Interface (CNI) represents the primary network interface that contemporary network solution providers implement to integrate with Kubernetes.

### Container Network Interface (CNI)

The Container Network Interface (CNI) is both a specification and a collection of libraries used to create network plugins for configuring network interfaces within Linux containers. Originating from the rkt network proposal, CNI has grown into an industry-standard framework adopted not only by Kubernetes but also by various other platforms, such as OpenShift, Mesos, Kurma, Cloud Foundry, Nuage, IBM, AWS EKS and ECS, and Lyft.

The CNI team maintains core plugins, but the ecosystem thrives with third-party contributions. Some notable plugins include:

- Project Calico: Provides a layer 3 virtual network for Kubernetes.
- Weave: Establishes a virtual network connecting multiple Docker containers across multiple hosts.
- Contiv networking: Offers policy-based networking.
- Cilium: Facilitates ePBF (enhanced Packet Filtering and Forwarding) for containers.
- Flannel: Delivers a layer 3 network fabric tailored for Kubernetes.

CNI plugins create a standardized networking interface, serving as a foundation for diverse networking solutions.

#### Container Runtime

CNI establishes a specification for network plugins used in application containers. However, these plugins need to be integrated into a container runtime, which offers certain services. In the context of CNI, an application container is an entity with its own IP address. For Docker, individual containers possess distinct IP addresses. In Kubernetes, each pod has its unique IP address, and the pod is recognized as the CNI container, with the containers within the pod being hidden from CNI.

The container runtime's responsibility involves configuring a network and subsequently executing one or more CNI plugins. These plugins are provided with network configuration in JSON format.

#### CNI plugin

The role of a CNI plugin involves creating a network interface within the container's network namespace and connecting the container to the host using a veth pair. Additionally, the plugin must assign an IP address using an IP address management (IPAM) plugin and establish routes.

In a CRI-compliant container runtime environment, the CNI plugin is executed as an executable, performing operations such as adding or removing a container from the network and reporting the version. The plugin operates through a simple command-line interface, standard input/output, and environment variables. Network configuration in JSON format is passed via standard input. A successful execution returns a zero exit code, and the generated interfaces are streamed in JSON format through standard output, particularly for the ADD command.

This straightforward interface is versatile, requiring no specific programming language or binary API. CNI plugin writers can use their preferred programming language.

Notably, the CNI specification can accommodate additional plugin-specific elements, as seen with the custom "bridge: cni0" element understood by the bridge plugin. The CNI specification also supports network configuration lists, enabling the invocation of multiple CNI plugins in sequence.

---

## Kubernetes and eBPF

Kubernetes boasts immense versatility and flexibility due to its developers' deliberate avoidance of assumptions and decisions that could limit future options. For instance, Kubernetes networking primarily operates at the IP and DNS levels, refraining from concepts like networks and subnets. Such aspects are delegated to networking solutions that interface with Kubernetes through broad but generic mechanisms like CNI. This approach fosters innovation by empowering implementers with a range of choices.

eBPF (extended Berkeley Packet Filter) emerges as a groundbreaking technology facilitating the secure execution of sandboxed programs in the Linux kernel without compromising system security or necessitating kernel modifications or kernel module alterations. These programs are triggered by events, profoundly impacting software-defined networking, observability, and security. Termed the "Linux super-power" by Brendan Gregg, ePBF significantly enhances networking capabilities.

While original BPF was limited to socket attachment for packet filtering, ePBF extends its reach to various objects, including Kprobes, tracepoints, network schedulers or qdiscs for classification or action, and XDP.

In traditional Kubernetes routing, kube-proxy plays a pivotal role. This user space process operates on each node, orchestrating iptable rules, and facilitating UDP, TCP, and STCP forwarding, as well as load balancing based on Kubernetes services. At a large scale, kube-proxy becomes a hindrance, with its sequential iptable rule processing and frequent user space to kernel space transitions causing unnecessary overhead. A promising alternative lies in an eBPF-based approach, capable of replacing kube-proxy more efficiently while fulfilling the same functions.

![eBPF](https://ebpf.io/static/diagram-b6b32006ea52570dc6773f5dbf9ef8dc.svg)

---

## Kubernetes networking solutions

### The Calico project

Calico stands as a versatile solution for virtual networking and network security within container environments. It seamlessly integrates with various prominent container orchestration frameworks and runtimes:

- Kubernetes (CNI plugin)
- Mesos (CNI plugin)
- Docker (libnetwork plugin)
- OpenStack (Neutron plugin)

Calico is adaptable for on-premises deployments or public cloud environments, offering its full array of features. Notably, Calico's network policy enforcement can be tailored for individual workloads, ensuring meticulous traffic control and dependable packet routing. It guarantees that data travels from its source to validated destinations. Calico also excels in translating network policy concepts from different orchestration platforms into its own network policy framework. In fact, Calico serves as the reference implementation for Kubernetes' network policy.

Furthermore, Calico is flexible enough to work in conjunction with Flannel. By leveraging Flannel's networking layer and integrating Calico's network policy capabilities, a comprehensive networking and security solution can be constructed.

### Weave Net

Weave Net simplifies networking for developers with a focus on ease of use and effortless configuration. It operates underneath using VXLAN encapsulation and employs micro DNS on each node. This approach elevates developers to a higher level of abstraction. They can assign names to containers, and Weave Net takes care of connecting them and facilitating the use of standard ports for services. This smooth transition aids in migrating existing applications into containerized environments and microservices.

Weave Net offers a CNI plugin designed for seamless integration with Kubernetes and Mesos. For Kubernetes versions 1.4 and above, integrating Weave Net is straightforward. A single command deploys a Daemonset that establishes connectivity with Weave Net. The Weave Net pods, distributed across all nodes, automatically handle the attachment of newly created pods to the Weave network.

In addition, Weave Net supports the network policy API, providing a comprehensive and user-friendly solution for network policy enforcement. This feature enhances security and control without sacrificing ease of setup.

To sum up, Weave Net streamlines networking complexities, enabling developers to work efficiently by naming containers and relying on Weave Net's seamless connectivity. It supports Kubernetes and Mesos integration and provides features like micro DNS and network policy support to enhance overall networking functionality.

### Cilium

Cilium, a project incubated by the Cloud Native Computing Foundation (CNCF), centers around eBPF-driven networking, security, and observability, complemented by its Hubble project.

#### Eﬃcient IP allocation and routing

Cilium offers diverse networking capabilities, supporting various models to facilitate seamless connectivity within and across clusters:

1. **Overlay Model:** Cilium enables a flat Layer 3 network structure that spans multiple clusters and links all application containers. It features encapsulation-based virtual networks utilizing formats like VXLAN, Geneve, and other Linux-supported formats. This model can be employed with virtually any network infrastructure, provided hosts have IP connectivity. It offers both flexibility and scalability.

2. **Native Routing Model:** In this approach, Kubernetes utilizes the regular routing table of the Linux host. Application containers' IP addresses must be routable within the existing network infrastructure. Native Routing mode is more advanced and requires familiarity with the underlying networking setup. It is well-suited for native IPv6 networks, cloud network routers, or when utilizing custom routing daemons.

#### Identity-based service-to-service communication

Cilium introduces a security management aspect that assigns a security identity to clusters of application containers governed by shared security policies. This identity becomes linked to all network packets originating from these containers. Through this approach, Cilium enables the validation of identity upon packet reception at the destination node. The management of security identities is facilitated through a key-value store, streamlining the secure and efficient administration of identities within the Cilium networking framework.

#### Load balancing

Cilium offers a distributed load balancing solution for managing traffic between application containers and external services, serving as an alternative to kube-proxy. This load balancing functionality employs efficient eBPF-powered hashtables, a more scalable approach compared to traditional iptables techniques. Cilium ensures high-performance load balancing while optimizing the utilization of network resources.

Cilium stands out in east-west load balancing by performing streamlined service-to-backend translation within the Linux kernel's socket layer. This eliminates the need for per-packet Network Address Translation (NAT) operations, leading to reduced overhead and enhanced performance.

For north-south load balancing, Cilium's eBPF implementation is finely tuned for maximum performance. It seamlessly integrates with eXpress Data Path (XDP) and supports advanced load balancing techniques like Direct Server Return (DSR) and Maglev consistent hashing. These capabilities enable efficient offloading of load balancing tasks from the source host, contributing to superior performance and scalability.

#### Bandwidth management

Cilium introduces effective bandwidth management by utilizing efficient Earliest Departure Time (EDT)-based rate-limiting with eBPF, specifically for egress traffic. This approach notably minimizes transmission tail latencies for applications, leading to improved performance.

#### Observability

Cilium offers extensive event monitoring enriched with valuable metadata. Alongside capturing source and destination IP addresses of dropped packets, it provides intricate label details for both senders and recipients. This metadata empowers advanced visibility and troubleshooting capabilities. Cilium also exports metrics through Prometheus, enabling seamless monitoring and analysis of network performance.

For heightened observability, the Hubble observability platform offers additional functionalities like service dependency maps, operational monitoring, alerting, and comprehensive insights into application and security aspects. Through flow logs, Hubble empowers administrators to gain valuable insights into the interactions and behaviors of services within the network.

---

## Using network policies

The Kubernetes network policy revolves around managing network traffic to specific pods and namespaces. In the context of a Kubernetes environment with numerous microservices orchestrated, effective management of networking and pod connectivity is vital. It's crucial to note that this policy isn't primarily a security mechanism. If an attacker can access the internal network, they can likely create pods that adhere to the existing network policy and communicate freely with other pods. It's important to recognize the strong interplay between the chosen networking solution and the implementation of network policy atop it.

### Understanding the Kubernetes network policy design

A network policy in Kubernetes establishes communication regulations for pods and other network endpoints in the cluster. Utilizing labels, it identifies specific pods and enforces whitelist rules to govern traffic access to those pods. These rules work alongside the namespace-level isolation policy and enable supplementary traffic based on defined criteria. Through network policy configuration, administrators gain the ability to refine and limit communication between pods, enhancing security and segmenting the network within the cluster.

### Network policies and CNI plugins

Network policies and CNI plugins have a complex interplay. Certain CNI plugins handle both network connectivity and network policies, while others address only one of these aspects. However, some plugins can work together, with one plugin managing connectivity and another handling policies. For instance, Calico and Flannel can collaborate in this manner. This dynamic interaction adds flexibility to how network policies are implemented within Kubernetes.

### Implementing network policies

The network policy API in Kubernetes is a standardized interface, but its implementation is closely linked to the networking solution in use. This entails having a specialized agent or gatekeeper on each node, like Cilium's eBPF-based kernel implementation. This agent performs the following tasks:

1. Intercepts incoming traffic to the node.
2. Validates the traffic against the defined network policies.
3. Approves or denies each request based on policy rules.

While Kubernetes offers the means to define and store network policies through its API, the actual enforcement of these policies is handed over to the networking solution or a dedicated network policy tool tightly integrated with the specific networking solution. For instance, Calico serves as an example of this approach, combining its networking and network policy solutions in a closely integrated manner.
