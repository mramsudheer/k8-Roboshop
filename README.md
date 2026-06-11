# Kubernetes Architecture <br>
![Acrhitecture Diagram](Architecture.jpg)
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
![MasterNode Diagram](MasterNode.jpg)
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
![Name Space Diagram](NameSpace1.jpg)
# NameSpace Shared Infra <br>
![Name Space shared Infra Diagram](NameSpace_SharedInfra.jpg)
![Name Space shared Infra Diagram](NameSpace_SharedInfra2.jpg)
**Kubernetes Networking** <br>
<https://medium.com/@h.stoychev87/kubernetes-networking-a-deep-dive-6081d794e97c> <br>
we’ll walk through the Kubernetes networking model, explore how traffic flows within and outside the cluster, break down key components like Services, CNI plugins, Ingress, network policies, kube-proxy modes, and finish with practical troubleshooting and exam tips. <br>
**The Kubernetes Networking Model** <br>
Kubernetes enforces three foundational networking principles:<br>

1.&emsp;Pod-to-Pod Communication: Every Pod can communicate directly with any other Pod across nodes, without NAT.<br>
2.&emsp;Node-to-Pod Communication: Nodes can reach every Pod, and Pods can reach nodes, also without NAT.<br>
3.&emsp;Pod IP Consistency: A Pod’s self-IP is identical to what other Pods see externally.<br>
This creates a flat, routable L3 network in which each Pod is a first-class network entity.<br>

•&emsp;Applications can communicate using standard IPs; no port translation is needed. <br>
•&emsp;Simplifies microservices communication.<br>
•&emsp;Enables network policies to be applied at the Pod level.<br>
**2.&emsp;The Five Types of Kubernetes Networking<br>**
Kubernetes networking addresses four concerns:<br>

1.&emsp;Container-to-Container Networking <br>
2.&emsp;Pod-to-Pod Networking<br>
3.&emsp;Pod-to-Service Networking<br>
4.&emsp;External/Internet-to-Service Networking (Ingress)<br>
5.&emsp;Pod-to-External Networking (Egress)<br>