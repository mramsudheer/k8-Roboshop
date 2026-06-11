# Kubernetes Architecture
===================================================================== <br>
![Acrhitecture Diagram](Architecture.jpg)
# Master Node <br>
![MasterNode Diagram](MasterNode.jpg)
Control Plane Roles Relative to Namespaces <br>
•&emsp;`kube-apiserver:` The absolute gatekeeper. Every single kubectl request passes here first. When you query a specific namespace, it evaluates your authorization permissions against that unique logical boundary before processing the request.<br>
•&emsp;`etcd:` The single source of truth database. It holds the ultimate structural state configuration for the entire system, mapping exactly which namespace tag belongs to each individual resource definition.<br>
•&emsp;`kube-scheduler:` The matchmaker. It scans for newly configured workloads via the API server and assigns them to an open Data Node based on hardware capacity. The scheduler does not isolate workloads physically by namespace unless you explicitly command it to via affinity policies.<br>
•&emsp;`kube-controller-manager:` The core driver loop. It continuously compares your cluster's current running state against your desired configuration (e.g., ensuring 3 replicas of a production pod remain functional). It operates globally but processes resources namespace-by-namespace. <br>
•&emsp;`cloud-controller-manager (CCM):` The external bridge. It links your cluster to cloud provider APIs (like AWS, Azure, or GCP). For example, when you provision a namespaced `Service` of type `LoadBalancer`, the CCM communicates outward to spin up a physical cloud balancer matching that isolated workload. <br>
# Name Space <br>
![Name Space Diagram](NameSpace1.jpg)
![Name Space Diagram](NameSpace1.jpg)
# NameSpace Shared Infra <br>
![Name Space shared Infra Diagram](NameSpace_SharedInfra.jpg)
![Name Space shared Infra Diagram](NameSpace_SharedInfra1.jpg)