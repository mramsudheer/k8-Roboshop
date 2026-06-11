# Kubernetes Architecture <br>
![Acrhitecture Diagram](images/Architecture.jpg)
**Core Architecture Components Explained** <br>
**1.&emsp; Control Plane (Master Node) Components**<br>
•&emsp;`kube-apiserver:` The front end for the control plane. It processes all inner and outer requests, intercepts traffic, and verifies permissions before modifying the cluster state. <br>
•&emsp;`etcd:` A highly available, secure, and distributed key-value store. It serves as Kubernetes' backup database, holding all configuration data and current state logs. <br>
•&emsp;`kube-scheduler:` The routing brain. It watches for newly created pods that have no assigned node and selects the healthiest physical worker node for them based on CPU, memory, and policy constraints.<br>
•&emsp;`kube-controller-manager:` The operator loop. It runs daemon controllers that constantly track the cluster status (e.g., Node Controller, Job Controller, Deployment Controller) and works to bring the current state closer to your desired state. <br>
•&emsp;`cloud-controller-manager (CCM):` The abstraction layer that decouples your internal cluster logic from external cloud platform services, auto-provisioning physical infrastructure assets like cloud-provider storage volumes or load balancers. <br>
**2. Worker Node Components** <br>
•&emsp;`kubelet:` An agent that runs on every single worker node in the cluster. It registers the node with the master control plane and ensures that the specific containers described in PodSpecs are up, healthy, and running.<br>
•&emsp;`kube-proxy:` A network agent running on each node that maintains network rules on host operating systems. It allows network communication to your pods from inside or outside the cluster.<br>
•&emsp;`Container Runtime:` The underlying software engine responsible for actually running the isolated container processes (e.g., containerd, CRI-O, or Docker Engine).<br>
# Master Node <br>
![MasterNode Diagram](images/MasterNode.jpg)
**Control Plane Roles Relative to Namespaces** <br>
•&emsp;`kube-apiserver:` The absolute gatekeeper. Every single `kubectl` request passes here first. When you query a specific namespace, it evaluates your authorization permissions against that unique logical boundary before processing the request.<br>
•&emsp;`etcd:` The single source of truth database. It holds the ultimate structural state configuration for the entire system, mapping exactly which namespace tag belongs to each individual resource definition.<br>
•&emsp;`kube-scheduler:` The matchmaker. It scans for newly configured workloads via the API server and assigns them to an open Data Node based on hardware capacity. The scheduler does not isolate workloads physically by namespace unless you explicitly command it to via affinity policies.<br>
•&emsp;`kube-controller-manager:` The core driver loop. It continuously compares your cluster's current running state against your desired configuration (e.g., ensuring 3 replicas of a production pod remain functional). It operates globally but processes resources namespace-by-namespace. <br>
•&emsp;`cloud-controller-manager (CCM):` The external bridge. It links your cluster to cloud provider APIs (like AWS, Azure, or GCP). For example, when you provision a namespaced `Service` of type `LoadBalancer`, the CCM communicates outward to spin up a physical cloud balancer matching that isolated workload. <br>
**Role-Based Access Control (RBAC)** <br>
To secure your shared multi-tenant cluster (dev, qa, prod), you need to enforce boundaries at two levels: `Access Isolation` using Role-Based Access Control (RBAC) and `Resource Isolation` using Resource Quotas. <br>By default, the Master Node control plane components (`kube-apiserver`, `etcd`, `kube-scheduler`) are entirely locked down and accessible only by cluster administrators or core system processes. Developers interact *only* with the `kube-apiserver`. <br>
**1.&emsp; Restricting Developer Access via RBAC**<br>To prevent a developer working in the dev namespace from viewing or altering things in the prod namespace, you must define a Role (what can be done) and a RoleBinding (who can do it) scoped strictly to that namespace.<br>**Step A: Create a Namespace-Scoped Role**<br>This configuration grants permission to read and write application workloads, but restricts the user from altering administrative cluster infrastructure or crossing over to other namespaces. <br>
```hcl
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev # Boundary limited strictly to the dev namespace
  name: developer-workload-manager
rules:
- apiGroups: [""] # Core API group
  resources: ["pods", "services", "configmaps", "secrets", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"] # Apps API group (Deployments, StateFulSets)
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```
**Step B: Bind the Role to a User or Group**<br>
This connects your specific developer identity to the namespace role created above. <br>
```hcl
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-compute-quota
  namespace: dev # Placed directly inside the dev namespace
spec:
  hard:
    # Compute Restrictions
    requests.cpu: "4"         # Total CPU requests allowed across all dev pods
    requests.memory: 8Gi      # Total Memory requests allowed
    limits.cpu: "8"           # Max absolute CPU spike ceiling
    limits.memory: 16Gi       # Max absolute Memory spike ceiling
    
    # Object Count Restrictions
    pods: "10"                # Max number of concurrent running pods
    services: "5"             # Max services allowed
    persistentvolumeclaims: "4" # Max storage claims allowed

```
# Name Space <br>
![Name Space Diagram](images/NameSpace1.jpg)
# NameSpace Shared Infra <br>
![Name Space shared Infra Diagram](images/NameSpace_SharedInfra.jpg)
![Name Space shared Infra Diagram](images/NameSpace_SharedInfra2.jpg)
**Kubernetes Networking** <br>
<https://medium.com/@h.stoychev87/kubernetes-networking-a-deep-dive-6081d794e97c> <br>
we’ll walk through the Kubernetes networking model, explore how traffic flows within and outside the cluster, break down key components like Services, CNI plugins, Ingress, network policies, kube-proxy modes, and finish with practical troubleshooting and exam tips. <br><br>
**The Kubernetes Networking Model** <br><br>
Kubernetes enforces three foundational networking principles:<br>

1.&emsp;Pod-to-Pod Communication: Every Pod can communicate directly with any other Pod across nodes, without NAT.<br>
2.&emsp;Node-to-Pod Communication: Nodes can reach every Pod, and Pods can reach nodes, also without NAT.<br>
3.&emsp;Pod IP Consistency: A Pod’s self-IP is identical to what other Pods see externally.<br> <br>
This creates a flat, routable L3 network in which each Pod is a first-class network entity.<br>

•&emsp;Applications can communicate using standard IPs; no port translation is needed. <br>
•&emsp;Simplifies microservices communication.<br>
•&emsp;Enables network policies to be applied at the Pod level.<br><br>
**2.&emsp;The Five Types of Kubernetes Networking**<br><br>
Kubernetes networking addresses four concerns:<br>

1.&emsp;Container-to-Container Networking <br>
2.&emsp;Pod-to-Pod Networking<br>
3.&emsp;Pod-to-Service Networking<br>
4.&emsp;External/Internet-to-Service Networking (Ingress)<br>
5.&emsp;Pod-to-External Networking (Egress)<br><br>
**3.&emsp;Communication Type: Container-to-Container (Inside a Pod)** <br>
•&emsp;`Shared Network Namespace:` Containers within the same pod share the same network namespace, meaning they can communicate with each other via localhost and share the same IP address and port space. <br>
•&emsp;`Inter-Process Communication (IPC):` Containers in a pod can use standard IPC mechanisms like SystemV semaphores or POSIX shared memory to communicate.<br>
•&emsp;`Shared Volumes:` Containers in the same pod can also communicate by reading and writing to shared volumes.<br>
Example:<br>
```hcl
kubernetes_node:
  name: "node-1"
  components:
    pod:
      name: "example-pod"
      ip: "10.244.1.5"
      network_namespace:
        interfaces:
          loopback:
            name: "lo"
            ip: "127.0.0.1"
            purpose: "Container-to-container localhost communication"
          ethernet:
            name: "eth0"
            ip: "10.244.1.5"
            purpose: "Pod network interface for all external communication"
        containers:
          - name: "container-1"
            role: "application"
            image: "nginx"
            listens_on:
              interface: "lo"
              port: 80
          - name: "container-2"
            role: "sidecar"
            action: "curl localhost:80"
            connects_to:
              target: "container-1"
              via: "127.0.0.1"
              protocol: "HTTP"
              description: "Sends GET / requests over the loopback interface"
```
> 📌 NOTE:The BusyBox sidecar continuously sends traffic to the Nginx container using `localhost:80.` <br><br>
![Name Space shared Infra Diagram](images/Container2ContainerNetwork.jpg)
Explanation: <br>
Inside every Kubernetes Pod, and on the host node, you will find several key network interfaces. Each one has a specific purpose in how Pods communicate internally and externally.<br>

Below is an explanation of each:<br>

>> Loopback Interface (lo)<br>

The loopback interface is a virtual network interface inside the Pod.<br>

•&emsp;Allows containers inside the Pod to communicate using localhost (127.0.0.1)<br>
•&emsp;Used for container-to-container communication within the same Pod<br>
•&emsp;Exists in every Linux network namespace<br>
> 📌 NOTE: Anything listening on `127.0.0.1` inside a Pod is accessible only to containers in that same Pod. <br><br>
>> Pod Interface (eth0)<br>

The main network interface inside the Pod.<br>
Every Pod gets one IP address, and that IP is assigned to its eth0 interface.<br>

&emsp;•&emsp;Sends/receives traffic between Pods <br>
&emsp;•&emsp;Communicates with Services, DNS, and external networks<br>
&emsp;•&emsp;Source IP for all traffic leaving the Pod<br><br>
📌 `eth0` holds the Pod IP (e.g., 10.244.1.5).<br>

>> VETH Pair (Virtual Ethernet Pair) <br>

A veth pair is like a virtual cable with two ends:<br>

&emsp;•&emsp;One end stays inside the Pod<br>
&emsp;•&emsp;One end stays on the host (node)<br><br>
📌 It is used to connect the Pod’s network namespace to the host networking system.<br>

Kubernetes CNIs (Calico, Flannel, Cilium, etc.) rely on veth pairs to plug Pods into the cluster network. <br>

>> VETH_P — Pod-Side VETH Interface<br>

The Pod-side end of a veth pair.<br>

&emsp;•&emsp;Connects the Pod to the host<br>
&emsp;•&emsp;Passes traffic from the Pod to the host networking stack<br>
&emsp;•&emsp;Usually named something like `eth0@if123` or similar internally<br><br>
📌 Inside the Pod, traffic exits via `eth0`, but this physically maps to the Pod-side veth interface.<br>

>> VETH_H — Host-Side VETH Interface<br>

The host/node side of the veth pair.<br>

&emsp;•&emsp;Bridges the Pod’s traffic to the CNI plugin (e.g., cni0 bridge)<br>
&emsp;•&emsp;Participates in the node’s routing and CNI overlays<br>
&emsp;•&emsp;Connects Pods to each other across nodes<br>
📌 This is where the Pod attaches to the node network.<br><br>
Its traffic then goes to the CNI bridge (`cni0` or `flannel.1`, etc.) depending on your CNI. <br><br>
Summary
| **Component** | **Location** | **Purpose** | **Example** |
| :--- | :--- | :--- | :--- |
| **lo (loopback)** | Inside Pod |	localhost connectivity between containers |	127.0.0.1 |
| **eth0** | Inside Pod |	Pod’s main network interface |	10.244.1.5 |
| **VETH_P** |	Pod | Pod-side end of veth pair | Connects to host |
| **VETH_H** |	Host |	Host-side end of veth pair | Connects to CNI bridge |

```hcl
Container(s) inside Pod
   |
   |  localhost ↔ lo (127.0.0.1)
   |
   |  external traffic ↔ eth0 (Pod IP)
                     ↕
                VETH_P (Pod end)
                     ↕
                VETH_H (Host end)
                     ↕
                  cni0 (bridge)
```
**4. Communication Type: Pod-to-Pod** <br><br>

&emsp;**IP-Per-Pod Model** <br> <br>
![Pod2PodNetwork Diagram](images/Pod2PodNetwork.jpg)
Each pod in Kubernetes is assigned a unique IP address, allowing direct communication between pods without the need for *Network Address Translation (NAT)*. This simplifies networking and ensures that pods can easily find and talk to each other across the cluster. <br><br>
&emsp;**Kube-proxy** <br> <br>
![kube-proxy Diagram](images/kube-proxy.jpg)
<br>This component runs on each node and manages network rules to allow communication between pods. It handles routing and load balancing for services within the cluster.<br><br>
&emsp;•&emsp;Watches the Kubernetes API server for changes to Services and Endpoints<br>
&emsp;•&emsp;Updates network rules (iptables, IPVS, or nftables) on each node to route traffic correctly<br>
&emsp;•&emsp;Load balances traffic across the pods backing a Service<br>
&emsp;•&emsp;Maintains the virtual IP (ClusterIP) routing so traffic to a Service IP reaches the right pods<br><br>
When you create a Service, kube-proxy ensures that traffic sent to the Service’s ClusterIP gets forwarded to one of the healthy pod IPs behind it. It doesn’t actually proxy the traffic itself in most modes — it sets up rules so the kernel handles the routing directly.<br>

**Three modes:**<br>

&emsp;1.&emsp;iptables mode (most common, still widely used) — Uses iptables rules for routing<br>
&emsp;2.&emsp;IPVS mode — Uses IPVS for better performance with many Services<br>
&emsp;3.&emsp;userspace mode (legacy) — Actually proxies connections in userspace
&emsp;**Тerminology explanation**<br><br>
>> cni0 Bridge<br>

The A `cni0` bridge is a virtual network bridge on the host node.<br>

&emsp;•&emsp;Acts like a virtual switch connecting all Pods on the same node.<br>
&emsp;•&emsp;Receives traffic from the host-side veth interface (VETH_H) of each Pod.<br>
&emsp;•&emsp;Allows same-node Pod-to-Pod communication at Layer 2 (no routing needed).<br>
&emsp;•&emsp;Connects to the node routing system for cross-node traffic.<br><br>
📌 *Think of it as the local hub where Pods “plug in” to communicate with each other or reach the node network.*<br>

>> CNI Overlay (VXLAN/IPIP)<br>

When Pods on different nodes need to talk, traffic leaves the cni0 bridge and is encapsulated by the CNI overlay:<br>

&emsp;•&emsp;VXLAN or IPIP wraps the Pod-to-Pod packet inside a new outer packet.<br>
&emsp;•&emsp;Outer IP addresses are the node IPs, inner IP addresses are the Pod IPs.<br>
&emsp;•&emsp;Sent over the physical network between nodes.<br>
&emsp;•&emsp;Decapsulated on the destination node and delivered to the target Pod via that node’s bridge.<br><br>
📌 *This is how Kubernetes achieves a flat, cluster-wide network, making all Pod IPs reachable across nodes.*<br>

>> Routing Table<br>

Every node has a routing table that decides where packets go:<br><br>
&emsp;•&emsp;Determines whether a Pod IP is local (on the same node) or remote (on another node).<br>
&emsp;•&emsp;Local Pod traffic is sent directly via the cni0 bridge.<br>
&emsp;•&emsp;Remote Pod traffic is sent to the CNI overlay interface (e.g., tunl0 or VXLAN).<br>
&emsp;•&emsp;Ensures packets are properly encapsulated and routed to the destination node.<br><br>
📌 *The routing table is what makes Kubernetes networking transparent to Pods — a Pod only sees a flat IP space, even if its peer is on another node.* <br>

>> iptables<br>

The legacy mechanism kube-proxy uses to implement Kubernetes Services.<br><br>

&emsp;•&emsp;Programs a large set of firewall rules in the Linux kernel.<br>
&emsp;•&emsp;Uses NAT (masquerading) and DNAT to redirect Service IPs (ClusterIPs) to backend Pod IPs.<br>
&emsp;•&emsp;Works by evaluating rules sequentially, which can become slow at scale.<br>
&emsp;•&emsp;Handles connection tracking to ensure packets of the same flow always reach the same Pod.<br>
&emsp;•&emsp;Used by default in many older Kubernetes setups.<br><br>
Think of `iptables` as a long chain of "if-this-then-that" traffic rules.<br>
Powerful, but as the cluster grows, the rule list becomes huge — and that slows things down.<br><br>

>> IPVS<br>

A faster, more scalable alternative to iptables for kube-proxy.<br><br>

&emsp;•&emsp;Implements L4 load balancing directly in the Linux kernel.<br>
&emsp;•&emsp;Uses hashed or round-robin algorithms to distribute traffic across Pods.<br>
&emsp;•&emsp;Builds a dedicated load-balancing table instead of long sequential rule lists.<br>
&emsp;•&emsp;More efficient for large clusters with many Services and Endpoints.<br>
&emsp;•&emsp;Supports health checking and connection persistence.<br><br>
Think of `IPVS` as a purpose-built, kernel-level load balancer:<br>
&emsp;•&emsp;Fast lookups, smarter algorithms, and far more efficient than traversing thousands of iptables rules.<br> <br>
&emsp;**Summary Diagram** <br>
```hcl
Container(s) inside Pod
   |
   |  localhost ↔ lo (127.0.0.1)
   |
   |  external traffic ↔ eth0 (Pod IP)
                     ↕
                VETH_P (Pod-side)
                     ↕
                VETH_H (Host-side)
                     ↕
                  cni0 (bridge)
                     ↕
         Routing Table + Overlay (VXLAN/IPIP)
                     ↕
          Node Network → other nodes → remote Pod
```
**5.&emsp;Communication Type: Pod-to-Service**<br><br>
Because Pods are ephemeral, Services provide a stable virtual IP (ClusterIP).<br><br>
![pod_to_service Diagram](images/pod_to_service.jpg)
&emsp;**How it works:** <br>

&emsp;1.&emsp;A client Pod resolves my-service via DNS → gets ClusterIP.<br>
&emsp;2.&emsp;Traffic is sent to the ClusterIP.<br>
&emsp;3.&emsp;kube-proxy intercepts and rewrites the packet to a real Pod IP.<br>
&emsp;4.&emsp;The packet reaches the backend Pod.<br>
&emsp;5.&emsp;Response is returned transparently.<br><br>
Summary Diagram <br>
```hcl
Client Pod
   |
   |  curl my-service → ClusterIP (virtual IP)
                     ↕
                  kube-proxy
                     ↕
          Load-balancing & DNAT to backend Pod IPs
                     ↕
           Backend Pod(s) (real Pod IPs)
```
This keeps Pod-to-Service communication simple for applications: Pods just use a single, stable IP, while kube-proxy handles the routing and load balancing behind the scenes.<br><br>
Example:<br>
```hcl
# ---------------------------
# Service
# ---------------------------
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
  clusterIP: 10.96.0.100
```

**6.&emsp;Kubernetes Services**<br><br>
Here is the right moment to discuss Kubernetes Services in more detail.<br><br>
A Service in Kubernetes is an abstraction that defines a logical set of Pods and a policy to access them. It provides a stable network endpoint (IP and DNS name) for Pods that may come and go.<br><br>
Pods in Kubernetes are ephemeral — they can be created, destroyed, or replaced. Their IP addresses change dynamically. Services solve this by giving a consistent way to access Pods.<br><br>
![services Diagram](images/services.jpg)<br>
Service types define how the service is exposed, as detailed in the official Kubernetes Service documentation.<br><br>
Explanation:<br>
&emsp;•&emsp;Service is an API resource: It exists in the Kubernetes API as a definable object that you can `kubectl apply`, `kubectl get`, or manage via the API server.<br><br>
&emsp;•&emsp;Service is also an object: When created, it becomes an actual object in etcd, and the Kubernetes control plane manages its lifecycle.<br><br>
So you can think of it as an API resource that represents a Service object in the cluster.<br><br>
&emsp;`kubectl get service my-service` <br>
Here service is the API resource type, and my-service is the object instance.<br><br>
📌 *Summary: Service = API resource type + object instance.* <br>

Service yaml code:<br>
```hcl
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80        # Service port
      targetPort: 8080 # Pod port
  type: ClusterIP
```
**Stable Cluster IP — ClusterIP:**<br>

&emsp;•&emsp;Each Service gets a fixed IP (ClusterIP) inside the cluster, which does not change, even if Pods are replaced.<br><br>
**Load Balancing:**<br>
&emsp;•&emsp;Services automatically distribute traffic to the backend Pods using `kube-proxy` (via iptables or IPVS).<br><br>
**Service Selector:**<br>
&emsp;•&emsp;The Service selects which Pods to send traffic to using labels. For example, `app=nginx`.<br><br>
**Service Types** <br><br>
![ServiceTypes Diagram](images/ServiceTypes.jpg)<br>
There are several types of Services, each suited for different use cases:<br><br>
&emsp;•&emsp;`ClusterIP:` Exposes the Service on an internal IP within the cluster. This is the default type and is used for communication between services within the cluster.<br>
&emsp;•&emsp;`NodePort:` Exposes the Service on each node’s IP at a static port. This allows external traffic to access the Service.<br>
&emsp;•&emsp;`LoadBalancer:` Exposes the Service externally using a cloud provider’s load balancer.<br>
&emsp;•&emsp;`ExternalName/External Service:` Maps the Service to the contents of the externalName field (e.g., my-app.example.com), allowing you to proxy to an external service.<br><br>
Use the right Service type<br><br>
`externalTrafficPolicy:` Local to preserve source IP and reduce hops:<br>
```hcl
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  externalTrafficPolicy: Local
  selector:
    app: my-app
  ports:
  - port: 80
  ```
**Practical Troubleshooting Commands: Service**<br>
Service Issues:<br>
```hcl
    kubectl get endpoints my-service
    kubectl describe svc my-service
    kubectl logs -n kube-system <kube-proxy-pod>
```
**Practical Troubleshooting Commands: Pod Networking Troubleshooting** <br>
Pod Troubleshoot:<br>
```hcl
    kubectl get pod my-pod -o wide
    kubectl exec pod-a -- ping <pod-b-ip>
    kubectl exec pod-a -- nslookup my-service
    kubectl exec my-pod -- ip addr
```
Pod Cannot Reach Service: <br>
```hcl
    kubectl get svc my-service
    kubectl get endpoints my-service
    kubectl get pods -l app=my-app
    kubectl describe svc my-service
```
Most common causes:<br>

&emsp;•&emsp;Service selector mismatch<br>
&emsp;•&emsp;Pods not Ready<br>
&emsp;•&emsp;kube-proxy misconfiguration<br>