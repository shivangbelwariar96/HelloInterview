# Facebook-like System Architecture on Kubernetes: A Complete Flow

## Introduction

Okay, let's build a detailed, step-by-step picture of how a user request flows through a large, Facebook-like system running on Kubernetes at a production level, incorporating all the specified components. This is indeed a complex scenario, typical of a high-scale, modern microservices architecture.

We'll trace the journey from the user's browser/app back to the data stores and internal services, and then the response journey back to the user. We'll integrate Kubernetes components, HashiCorp tools, monitoring, and CI/CD throughout the explanation.

## Scenario

A user opens their Facebook-like app/website and wants to see their Newsfeed.

### Assumptions

- The system is deployed across multiple geographical regions for resilience and performance.
- Microservices architecture is strictly followed.
- Kubernetes is the orchestrator for the application services and internal infrastructure components.
- Cloud provider infrastructure is utilized (VMs for K8s nodes, managed databases where applicable, persistent storage).

## The High-Level Architecture

Before diving into the steps, let's visualize the layers:

- **Client**: User's device (Browser, Mobile App).
- **CDN (Content Delivery Network)**: Serves static assets (images, CSS, JS).
- **Global Load Balancers**: Distribute traffic to regional clusters.
- **Regional Load Balancers**: Distribute traffic within a region to the API Gateways.
- **API Gateways**: Entry point into the cluster, handles initial request processing, routing, authentication checks, protocol translation.
- **Kubernetes Cluster**: Hosts all microservices, data store access layers, messaging queues, and supporting infrastructure.
- **Microservices**: Application logic (Auth, Newsfeed, Post, Friend, Message, etc.).
- **Data Stores**: Databases (MySQL, Cassandra), Caching (Redis), Message Queue (Kafka).
- **Supporting Tools**: Service Discovery (Consul/K8s Service), Secrets Management (Vault), Monitoring (Prometheus, Grafana, Splunk), CI/CD (Jenkins, Helm).

## Step-by-Step Request Flow & Kubernetes Integration

Let's trace the "Fetch Newsfeed" request:

### Step 1: User Initiates Request (Client)

- The user opens the app or website. The client application (running on their device) sends an HTTP/S request to fetch the latest Newsfeed data.
- **Protocols**: HTTP/S for initial communication. WebSocket might be used later for real-time updates (like new messages or likes appearing instantly).

### Step 2: CDN Check (for Static Assets)

- If the request was for static content (images, CSS, JS needed to render the newsfeed page), the client first checks the nearest CDN edge location.
- **CDN**: Serves the static content directly, reducing load on the backend and latency for the user. The actual Newsfeed data request bypasses the CDN or is handled by the origin server after the initial page load.

### Step 3: Global Load Balancing

- The request for dynamic Newsfeed data (e.g., https://www.facebook.com/api/v1/newsfeed) hits a Global Load Balancer (e.g., using DNS-based routing like AWS Route 53 weighted routing, or a dedicated global LB service).
- **Global LB**: Directs the user's request to the geographically nearest or least loaded Regional Load Balancer.

### Step 4: Regional Load Balancing

- The request arrives at a Regional Load Balancer (e.g., cloud provider's Application Load Balancer/Network Load Balancer, or a self-managed HAProxy/Nginx farm).
- **Regional LB**: This LB is configured to forward traffic to the API Gateway layer running within the regional Kubernetes cluster. It likely uses a Service of type LoadBalancer or NodePort exposed by the API Gateway Pods.

### Step 5: API Gateway (Ingress & Service)

- The request hits the API Gateway. This layer is crucial. It's typically implemented as one or more Deployments of API Gateway Pods (e.g., running Envoy, Nginx, or a custom gateway service).
- **Kubernetes**: The API Gateway Pods are managed by a Deployment ensuring a desired number of replicas are running. A Service of type ClusterIP (for internal routing) and potentially LoadBalancer/NodePort (exposed to the regional LB) provides a stable endpoint for the gateway Pods. An Ingress resource often configures the routing rules within the cluster, pointing the external request path (/api/v1/newsfeed) to the correct API Gateway Service.

**API Gateway Functions**:
- TLS Termination: Decrypts the HTTPS request.
- Initial Authentication/Session Check: Validates the user's session token or JWT. This might involve a quick call to the Auth Service.
- Rate Limiting: Protects backend services from excessive traffic.
- Request Validation: Basic checks on headers, request size.
- Protocol Translation: If a mobile app uses gRPC internally, the gateway might translate it to REST for backend services or vice-versa, or pass it through if the service supports gRPC directly.
- Routing: Based on the request path and headers, the Gateway decides which backend microservice should handle the request (in this case, the Newsfeed Service). This routing decision uses Kubernetes Service Discovery (via DNS or potentially Consul if used for a service mesh layer).

### Step 6: Authentication and Authorization (Auth Service)

- The API Gateway (or the Newsfeed Service itself, depending on the architecture pattern - often the Gateway handles initial auth) makes an internal call to the Auth Service.
- **Kubernetes**: The Auth Service runs as a Deployment of Pods, exposed via a Service (type ClusterIP).
- **Communication**: This internal call is likely gRPC for performance and efficiency, using the Auth Service's stable DNS name provided by Kubernetes Service Discovery (e.g., auth-service.my-namespace.svc.cluster.local).

**Auth Service Functions**:
- Validates the user's identity based on the token/cookie.
- Checks if the user is authorized to access the requested resource (e.g., their newsfeed).
- Returns user ID and permissions to the caller.

### Step 7: Request Routing to Newsfeed Service (Service Discovery)

- The API Gateway (or the next service in the chain) needs to send the request to the Newsfeed Service.
- **Kubernetes**: The Gateway uses the Service discovery mechanism. It resolves the DNS name of the Newsfeed Service (e.g., newsfeed-service.my-namespace.svc.cluster.local). The K8s DNS service (CoreDNS or Kube-DNS) returns the stable IP address of the Newsfeed Service.
- **Kubernetes Service (type ClusterIP)**: This IP is a virtual IP managed by Kube-proxy on each Node. Kube-proxy configures IPtables or IPVS rules to distribute connections arriving at the Service IP among the healthy Pods belonging to the Newsfeed Service Deployment. This is internal load balancing within the cluster.
- **Consul (Optional/Alternative)**: If Consul is used for service discovery alongside or instead of K8s DNS, the Gateway might query Consul to find healthy instances of the Newsfeed Service. Consul agents running on Nodes (as a DaemonSet) register and health check services.

### Step 8: Newsfeed Service Processing

- The request lands on one of the Newsfeed Service Pods. This Pod is running a Spring Boot application containing the Newsfeed logic.
- **Kubernetes**: Newsfeed Service Pods are managed by a Deployment. The HorizontalPodAutoscaler (HPA) is configured for this Deployment. It monitors metrics like CPU utilization, memory usage, or custom metrics (e.g., requests per second from Prometheus) and automatically increases or decreases the number of Newsfeed Pods (by scaling the underlying ReplicaSet) to handle varying load. This ensures auto-scalability.

**Newsfeed Service Logic**: To build the Newsfeed, the service needs data from other services and data stores. It will make calls to:
- Friend Service: To get a list of friends and people the user follows. Likely gRPC communication via the Friend Service's K8s Service IP/DNS.
- Post Service: To fetch recent posts from the user's friends and followed entities. Likely gRPC via Post Service's K8s Service.
- Like/Comment/Share Services: To get interaction counts for posts. Likely gRPC.
- User Profile Service: To get details about users/pages involved in posts. Likely gRPC.

**Inter-Service Communication**: These calls use the stable Service IPs/DNS names. NetworkPolicy is likely implemented in the namespace to restrict which Pods (e.g., only Newsfeed Pods) are allowed to connect to the Friend Service Pods, enhancing security.

### Step 9: Data Access (Databases & Cache)

The various services (Newsfeed, Post, Friend, etc.) need to retrieve and store data.

**MySQL (for structured data like User Profiles, Friend Relationships)**:
- Runs as StatefulSets in Kubernetes for stable network identity, ordered deployment/scaling, and persistent storage.
- Each MySQL instance in the cluster uses a PersistentVolumeClaim (PVC) to request storage. The PVC binds to a PersistentVolume (PV) provisioned from the underlying cloud storage (e.g., EBS volume). This ensures data persists even if a Pod is rescheduled or dies.
- Services connect to MySQL using connection pools within their Pods, connecting to the MySQL Service's stable IP/DNS (also managed by a K8s Service of type ClusterIP, often with headless Service for StatefulSets to allow direct pod addressing if needed).
- Replication and sharding are managed either within the MySQL StatefulSet setup or by external tooling, but the K8s layer provides the orchestration and persistent storage.

**Cassandra (for unstructured/high-write data like Posts, Messages)**:
- Runs as StatefulSets in Kubernetes.
- Each Cassandra node uses PVCs for persistent storage (PVs).
- Services connect to the Cassandra cluster via drivers that understand the cluster topology, connecting to the Cassandra Service's stable IP/DNS. Cassandra handles its own data distribution (sharding/replication) across its nodes/Pods.

**Redis (for Caching, Sessions, Pub/Sub)**:
- Runs as StatefulSets for persistence (if used as a durable cache/session store) or as a standard Deployment if used purely as an ephemeral cache.
- Uses PVCs if persistence is needed.
- Services connect to Redis via its K8s Service IP/DNS. Redis clustering might be managed internally or via external tools.

**Vault Integration for Database Credentials**:
- Services do not have database passwords hardcoded or stored in simple K8s Secrets.
- Instead, they use Vault. The Pods' ServiceAccounts are configured to authenticate with Vault.
- Vault Policies are defined, granting specific Vault Roles (bound to K8s ServiceAccounts) permission to read database credentials from dynamic secrets engines in Vault (e.g., Vault can generate temporary, high-privilege database credentials on demand).
- The application code or a Vault Agent/CSI Driver running as a sidecar container in the application Pod fetches the credentials from Vault securely at runtime. The Secret resource in K8s might be populated by the Vault CSI driver directly into the Pod's volume. This is a key part of the "FAANG level" security.

### Step 10: Asynchronous Processing (Kafka)

- Actions triggered by the Newsfeed (e.g., user likes a post displayed in the feed) might need asynchronous processing (sending notifications, updating analytics, triggering newsfeed recalculations for others).
- The service handling the action (e.g., Like Service) publishes a message (event) to a Kafka topic.
- **Kafka**: Runs as a StatefulSet in Kubernetes, using PVCs for persistent log storage. Zookeeper (also likely a StatefulSet) is used for coordination.
- Other services (e.g., Notification Service, Analytics Service) subscribe to relevant Kafka topics.
- **Kubernetes**: Subscriber services run as Deployments and are automatically scaled (HPA) based on the lag of their Kafka consumer groups.

### Step 11: Configuration Management (ConfigMap & Consul)

- Application services need configuration (database connection strings - though credentials come from Vault, topic names, feature flags, service endpoints).
- **ConfigMap**: Used for storing non-sensitive configuration data. Mounted as files or injected as environment variables into the Pods. Managed and versioned alongside Deployments using Helm.
- **Consul (Optional but common)**: Can be used for dynamic, runtime configuration updates via its K/V store. Services can watch Consul for changes without needing a Pod restart. Consul agents run as a DaemonSet or are deployed alongside applications.

### Step 12: Building the Response

- The Newsfeed Service compiles all the retrieved data (posts, likes, comments, user info).
- It formats the response (e.g., JSON).

### Step 13: Response Journey (Reverse Path)

- The Newsfeed Service Pod sends the response back to the API Gateway Pod (via K8s Service IP/DNS).
- The API Gateway potentially adds headers, performs final checks, and sends the response back to the Regional Load Balancer.
- The Regional Load Balancer sends it back to the Global Load Balancer.
- The Global Load Balancer sends it back to the user's client device.
- **Protocols**: Primarily HTTP/S. WebSocket might be used for subsequent real-time updates pushed from the server (e.g., a new comment appears without refreshing).

## Kubernetes Components Explained in Context

Let's revisit the K8s components and their specific roles in this system:

- **Pod**: The smallest deployable unit. Each instance of a microservice (Newsfeed, Auth, etc.), database node, or supporting agent runs as one or more containers within a Pod. They share network namespace and storage, allowing sidecars (like Vault Agent, logging agents).

- **ReplicaSet**: Ensures a stable set of replica Pods is running for a Deployment. If a Pod crashes, the ReplicaSet controller notices and creates a new one. Managed automatically by Deployments.

- **Deployment**: Declares the desired state for stateless applications (Newsfeed, Auth, API Gateway, etc.). Manages the creation and scaling of ReplicaSets. Handles rolling updates (deploying new versions with minimal downtime) and rollbacks.

- **StatefulSet**: Manages stateful applications (MySQL, Cassandra, Kafka, Redis if persistent). Provides stable network identifiers (sticky hostnames), stable persistent storage (tied to PVCs), and ordered, graceful deployment, scaling, and deletion. Essential for databases and message queues where identity and data persistence matter.

- **DaemonSet**: Ensures that all (or a subset) of Nodes run a copy of a Pod. Used for cluster-wide agents like log collectors (Splunk Forwarder agent), node monitoring agents (Node Exporter for Prometheus), or network plugins.

- **Job**: Represents a task that runs to completion. Used for one-off tasks like database schema migrations (executed before a new service version starts) or batch processing jobs (e.g., calculating user engagement stats periodically).

- **CronJob**: Schedules Jobs to run periodically (like cron). Used for automated daily/hourly tasks like data backups, report generation, or cleaning up old data.

- **Service**: An abstract way to expose a network application running on a set of Pods. Provides a stable IP address and DNS name regardless of Pod state or location. Kube-proxy implements the virtual IP and load balances traffic to healthy Pods. Crucial for both internal (ClusterIP) and external (LoadBalancer, NodePort) communication.

- **Ingress**: Manages external access to services within the cluster, typically HTTP/S. It provides URL-based routing, SSL termination, and name-based virtual hosting. An Ingress Controller (like Nginx, Envoy) running as a Deployment/StatefulSet implements the Ingress rules. Used here for the API Gateway's external exposure.

- **ConfigMap**: Stores non-confidential configuration data. Pods consume this data via environment variables or mounted files. Used for application settings, feature flags, general service configuration.

- **Secret**: Stores sensitive information (passwords, API keys, TLS certificates). Base64 encoded by default, but encrypted at rest in etcd. Can be consumed by Pods via environment variables or mounted files. Integrated with Vault for better security practices.

- **PersistentVolume (PV)**: A piece of storage in the cluster made available by an administrator (or dynamically provisioned). It's a cluster resource.

- **PersistentVolumeClaim (PVC)**: A request for storage by a user/Pod. A Pod needing persistent storage (like a database Pod in a StatefulSet) requests a PVC, which then binds to a suitable PV.

- **Namespace**: Provides a mechanism to scope resources (Pods, Services, Deployments, etc.) and provide logical isolation. Used to separate different environments (dev, staging, prod) or different applications/teams within the cluster.

- **Node**: A worker machine (VM or physical server) in the Kubernetes cluster where Pods are scheduled and run. Kubelet runs on each Node, managing Pods and communicating with the control plane.

- **ClusterRole / Role & ClusterRoleBinding / RoleBinding**: Kubernetes RBAC (Role-Based Access Control). Defines permissions (what actions can be performed on which resources).
  - **Role**: Defines permissions within a specific namespace.
  - **ClusterRole**: Defines permissions across the entire cluster or for cluster-scoped resources (Nodes, PersistentVolumes).
  - **RoleBinding**: Grants permissions defined in a Role to a user, group, or ServiceAccount within a specific namespace.
  - **ClusterRoleBinding**: Grants permissions defined in a ClusterRole to a user, group, or ServiceAccount across the entire cluster. Essential for securing access to K8s resources and for Vault integration (allowing ServiceAccounts to authenticate).

- **HorizontalPodAutoscaler (HPA)**: Automatically scales the number of Pod replicas (for Deployments, ReplicaSets, StatefulSets) based on observed metrics (CPU, Memory, or custom metrics from Prometheus). This is the primary K8s component enabling auto-scalability of application logic.

- **NetworkPolicy**: Specifies how groups of Pods are allowed to communicate with each other and external network endpoints. Implemented by the network plugin (CNI - Container Network Interface). Used to enforce network segmentation and security (e.g., ensuring only the API Gateway can talk to internal services, or limiting database access).

## HashiCorp Tooling in Production

### Vault:
- Runs as a StatefulSet in K8s for high availability and persistence.
- Centralized Secrets Management. Stores and manages database credentials, API keys, TLS certificates, etc.
- **Vault Policies**: Define granular rules about what secrets a user or machine identity (like a K8s ServiceAccount) can read, write, or manage.
- **Vault Roles**: Map authentication methods (like the Kubernetes ServiceAccount auth method) to specific Vault policies. A K8s ServiceAccount presents its token to Vault, and Vault verifies it with the K8s API server and grants the policies associated with the mapped role.
- Secrets are injected into Pods at runtime either by application code querying Vault API, a Vault Agent sidecar, or preferably the Vault CSI Driver, which mounts secrets directly into the Pod's filesystem as a volume, keeping secrets out of etcd and environment variables.

### Consul:
- Runs as a StatefulSet for servers and potentially a DaemonSet for clients on Nodes.
- **Service Discovery**: While K8s Service DNS provides basic discovery, Consul can offer more features, especially if integrating services outside the K8s cluster or using Consul's service mesh features (Consul Connect). Services can register themselves with Consul.
- **Configuration**: Consul's Key/Value store can be used for dynamic configuration that applications can watch for changes without restarts. A ConfigMap could be populated from Consul, or applications can query Consul directly.
- **Health Checking**: Consul performs detailed health checks on registered service instances.

## Helm for Deployment and Management

**Helm**: The package manager for Kubernetes.
- Each microservice or group of related services (including their required ConfigMaps, Secrets placeholders, Services, Deployments/StatefulSets, HPAs, NetworkPolicies, etc.) is packaged as a Helm Chart.
- The chart defines the Kubernetes manifests as templates.
- A values.yaml file (and environment-specific overlays like values-prod.yaml) holds configurable parameters like image versions, replica counts, resource requests/limits, service ports, ConfigMap data, etc.
- **Usage**: `helm install my-service ./my-service-chart -n my-namespace -f values-prod.yaml` or `helm upgrade my-release ./my-service-chart -n my-namespace -f values-prod.yaml`.
- Helm simplifies the deployment, versioning, and management of complex applications consisting of many K8s resources. It's integrated into the CI/CD pipeline.

## CI/CD Pipeline with Jenkins

**Jenkins**: Orchestrates the build, test, and deployment process.
- A Jenkins Pipeline (defined in a Jenkinsfile) is triggered by code commits.

**Pipeline Steps**:
1. **Build**: Compile Spring Boot application, run unit tests.
2. **Containerize**: Build a Docker image for the service.
3. **Scan**: Security scans on the code and Docker image.
4. **Push**: Push the Docker image to a central Container Registry.
5. **Vault Integration**: Jenkins job authenticates with Vault to fetch credentials needed for deployment (e.g., Kubeconfig credentials to access the K8s API, credentials to update Helm repositories). Using Vault policies and roles tied to a Jenkins Service Account in Vault.
6. **Helm Packaging**: Update the service's Helm chart values.yaml file with the new Docker image tag.
7. **Deployment**: Use Helm to deploy the updated chart to the target Kubernetes cluster/namespace (helm upgrade). Helm interacts with the K8s API server to create/update Deployments, Services, etc.
8. **Testing**: Run automated integration and end-to-end tests against the newly deployed version in the cluster.
9. **Monitoring/Alerting**: Check deployment health signals via Prometheus/Grafana dashboards. Jenkins might check for critical alerts in Splunk/Prometheus.
10. **Rollback**: If tests fail or monitoring indicates issues, Jenkins triggers a Helm rollback to the previous working version.

## Monitoring and Logging

### Prometheus:
- Runs as a StatefulSet or Deployment in K8s.
- Scrapes metrics from:
  - Kube-state-metrics (Deployment): Provides metrics about K8s objects themselves (Pod counts, PVC status, etc.).
  - Node Exporter (DaemonSet): Provides node-level metrics (CPU, memory, network I/O).
  - Application Pods: Spring Boot Actuator endpoints exposing Prometheus metrics, or custom application metrics libraries. K8s Service Discovery helps Prometheus find application targets.
  - Infrastructure components (MySQL, Cassandra, Redis exporters).
- Data is stored in Prometheus's time-series database.

### Grafana:
- Runs as a Deployment in K8s.
- Connects to Prometheus (and potentially other data sources like Splunk).
- Provides dashboards to visualize metrics (request rates, latency, error rates per service, resource utilization per Pod/Node, database performance). Used for real-time monitoring and historical analysis.

### Splunk (or alternative like Elastic Stack - Elasticsearch, Logstash, Kibana - ELK, or Loki):
- **Logging Agents**: A logging agent (e.g., Splunk Forwarder, Filebeat, Fluentd) runs as a DaemonSet on every K8s Node.
- Collects container logs from the Node's filesystem (/var/log/containers).
- Forwards logs to a centralized Splunk indexer cluster (running inside or outside K8s).
- Provides centralized log aggregation, searching, analysis, and alerting on application errors, warnings, security events, and user activity.

## Auto-Scalability in Action

The entire system is designed for auto-scalability at multiple layers:

### Infrastructure Layer (Cloud Provider):
- Kubernetes Nodes themselves run on VMs in a cloud provider's Autoscaling Groups.
- The Cluster Autoscaler K8s component monitors for pending Pods that cannot be scheduled due to insufficient resources (CPU, Memory) on existing Nodes. If necessary, it signals the cloud provider's Autoscaling Group to add more Nodes.
- Persistent Storage (PVs) is typically backed by scalable cloud storage volumes.

### Kubernetes Pod Layer:
- **HorizontalPodAutoscaler (HPA)**: The core of application auto-scaling. Monitors application-level metrics (CPU, Memory, or custom metrics from Prometheus like requests/second, queue size) and automatically adjusts the replicas count of Deployments, ReplicaSets, or StatefulSets. If CPU goes up, HPA adds more Pods; if it goes down, it removes Pods (gracefully).

### Data Layer:
- Databases (MySQL, Cassandra) are sharded horizontally. Adding more shards/nodes increases data capacity and query throughput. K8s StatefulSets facilitate scaling the number of database replicas/shards.
- Kafka scales by adding more brokers (StatefulSet instances) and partitioning topics.
- Redis scales via clustering.

## How Pods Communicate (Recap)

- **Within the same Pod**: Containers communicate via localhost or shared volumes.

- **Between Pods in the same Namespace/Cluster**:
  - The primary method is via Kubernetes Services.
  - A source Pod resolves the target Service's DNS name (e.g., newsfeed-service.my-namespace.svc.cluster.local).
  - The K8s DNS service returns the Service's stable ClusterIP.
  - The source Pod sends traffic to the ClusterIP.
  - Kube-proxy on the source Pod's Node (or the destination Pod's Node, depending on the IPtables/IPVS mode) intercepts traffic to the ClusterIP and uses its rules to forward it to the IP and Port of one of the healthy backend Pods belonging to that Service.
  - NetworkPolicy can restrict which connections are allowed at the IP/Port level between Pods.
  - Protocols like gRPC or REST are used over this network connection.
  - Consul (if used) can supplement or replace K8s DNS for discovery and health checks, potentially involving a service mesh sidecar pattern.

- **From External Traffic to Pods**:
  - External traffic hits Regional Load Balancers.
  - These LBs are configured to forward traffic to K8s NodePorts or a Service of type LoadBalancer.
  - An Ingress Controller running as Pods (exposed via a NodePort/LoadBalancer Service) receives the traffic.
  - The Ingress Controller's rules route the request to the appropriate internal Service (type ClusterIP).
  - The Service then load balances the request to the target application Pods.

## Conclusion

This describes a complex, highly available, scalable, and secure system architecture typical of a FAANG-level platform, leveraging Kubernetes as the core orchestration layer. Every component, from user request to data storage and back, is managed and scaled by Kubernetes primitives (Deployments, StatefulSets, Services, HPAs). HashiCorp Vault and Consul provide critical capabilities like secrets management and dynamic configuration, deeply integrated into the K8s workflow using Service Accounts, policies, and tools like the CSI driver. Helm streamlines the deployment process via CI/CD pipelines orchestrated by Jenkins. Robust monitoring and logging with Prometheus, Grafana, and Splunk provide the necessary visibility and alerting to operate such a large system effectively. This layered approach ensures resilience, performance, security, and manageability at scale.
