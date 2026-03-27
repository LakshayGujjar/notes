# Master Node Components

The master node contains several control plane components that manage the entire Kubernetes cluster. It keeps track of all nodes, decides where applications should run, and continuously monitors the cluster.

## ETCD

Kubernetes maintains detailed information about each container and its corresponding node in a highly available key-value store called etcd. Etcd uses a simple key-value format along with a quorum mechanism, ensuring reliable and consistent data storage across the cluster.

## Kubernetes scheduler

The scheduler takes into account current load, resource requirements, and specific constraints like taints, tolerations, or node affinity rules. This scheduling process is vital for efficient cluster operation.
Other key master node components include:
- ETCD Cluster: Stores cluster-wide configuration and state data.
- Kube Scheduler: Determines the best node for new container deployments.
- Controllers: Manage node lifecycle, container replication, and system stability.
- Kube API Server: Acts as the central hub for cluster communication and management.

---

# Worker Node Components

- **Kubelet:** Manages container lifecycle on an individual node. It receives instructions from the Kube API server to create, update, or delete containers, and regularly reports the node's status.
- **Kube Proxy:** Configures networking rules on worker nodes, thus enabling smooth inter-container communication across nodes. For instance, it allows a web server on one node to interact with a database on another.

- ContainerD, a CRI-compatible runtime, integrates directly with Kubernetes—eliminating the need for the Docker Shim. Originally bundled with Docker, ContainerD has evolved into a standalone project under the Cloud Native Computing Foundation. You can install and use it for basic container operations through CLI

- CRI CTL (crictl), designed to interact with any CRI-compatible container runtime, including ContainerD, Rocket, and others. Unlike CTR and NerdCTL—which are developed by the ContainerD community—crictl is maintained by the Kubernetes community and is primarily intended for debugging and inspection.

- etcd—a distributed, reliable key-value store that is both simple and fast.
you  an install etcd and perform following actions :

```
./etcdctl set key1 value1  [set key and value to be stored in etcd]
./etcdctl get key1  [retrival]
./etcdctl  [display all available subcommands]
./etcdctl --version [here latest API vesion should be API version 3] otherwise set following env var : export ETCDCTL_API=3
```

Any changes you make to the cluster—whether adding nodes, deploying pods, or configuring ReplicaSets—are first recorded in etcd. Only after etcd is updated are these changes considered to be complete.

### High Availability Considerations

In a production Kubernetes environment, high availability (HA) is paramount. By running multiple master nodes with corresponding etcd instances, you ensure that your cluster remains resilient even if one node fails.
To enable HA, each etcd instance must know about its peers. This is achieved by configuring the `--initial-cluster` parameter with the details of each member in the cluster.

---

## Kube API Server

when u run a kubectl command , The server processes this request by authenticating the user, validating the request, fetching data from the etcd cluster, and replying with the desired information.
It can also be installed individually.

---

## Kube Controller Manager

a controller acts like a department in an organization—each controller is tasked with handling a specific responsibility. For instance, one controller might monitor the health of nodes, while another ensures that the desired number of pods is always running. These controllers constantly observe system changes to drive the cluster toward its intended state.
The Node Controller, for example, checks node statuses every five seconds through the Kube API Server.
All individual controllers are bundled into a single process known as the Kubernetes Controller Manager. When you deploy the Controller Manager, every associated controller is started together. This unified deployment simplifies management and configuration.
This can also be installed individually.

---

## Kube Scheduler

The primary responsibility of the Kubernetes scheduler is to assign pods to nodes based on a series of criteria. This ensures that the selected node has sufficient resources and meets any specific requirements.

**1. Filtering Phase**
In the filtering phase, the scheduler eliminates nodes that do not meet the pod's resource requirements. For example, nodes that lack sufficient CPU or memory are immediately excluded.

**2. Ranking Phase**
After filtering, the scheduler enters the ranking phase. Here, it uses a priority function to score and compare the remaining nodes on a scale from 0 to 10, ultimately selecting the best match.
This can also be installed individually.

---

## Kubelet

The Kubelet is often described as the "captain of the ship." It oversees node activities by managing container operations such as starting and stopping containers based on instructions from the master scheduler. Additionally, the Kubelet registers the node with the Kubernetes cluster and continuously monitors the state of pods and their containers. It regularly reports the status of the node and its workloads to the Kubernetes API server.
This can also be installed individually.

---

## Kube Proxy

Kubernetes enables every pod within a cluster to communicate with one another by deploying a robust pod networking solution. This creates an internal virtual network that spans all nodes, connecting every pod.
Imagine your web application is running on one node while your database application is on another. Though the web application could connect to the database via its pod IP, these IPs are transient and may change. The recommended solution is to create a Service. By exposing the database through a Service (e.g., using the name "DB"), the web application can maintain a consistent connection without relying on fluctuating pod IPs. Each Service is assigned a stable IP address, and traffic routed to the Service is automatically forwarded to the appropriate backend pod.

Kube Proxy is a lightweight process that runs on every node in the Kubernetes cluster. Its key function is to monitor for Service creations and configure network rules that redirect traffic to the corresponding pods. One common method it uses is by setting up IP tables rules.
For example, if a Service is assigned the IP 10.96.0.12, Kube Proxy configures the IP tables on each node so that any traffic directed to that IP is forwarded to the actual pod IP, such as 10.32.0.15. This redirection mechanism ensures that Services work transparently across the cluster, regardless of which node initiates the request.
This can also be installed individually.

---

# PODS

A pod represents a single instance of an application and is the smallest deployable unit in Kubernetes.
Remember, scaling an application in Kubernetes involves increasing or decreasing the number of pods, not the number of containers within a single pod.
However, a pod can also contain multiple containers, which are usually complementary rather than redundant.
e.g. - you might include a helper container alongside your main application container to support tasks like data processing or file uploads.
the following command creates a pod that deploys an instance of the nginx Docker image, pulling it from a Docker repository :

```
kubectl run nginx --image nginx
kubectl get pods >> to get pods details
```

## A. PODS with YAML

Kubernetes leverages YAML files to define objects such as Pods, ReplicaSets, Deployments, and Services. These definitions adhere to a consistent structure, with four essential top-level properties: apiVersion, kind, metadata, and spec.

Every Kubernetes definition file must include the following four fields:

```yaml
apiVersion:
kind:
metadata:
spec:
```

**1. apiVersion**
This field indicates the version of the Kubernetes API you are using. For a Pod, set apiVersion: v1

**2. kind**
This specifies the type of object being created.
E.G. kind: Pod. Other objects might include ReplicaSet, Deployment, or Service.

**3. metadata**
The metadata section provides details about the object, including its name and labels. It is represented as a dictionary. It is essential to maintain consistent indentation for sibling keys to ensure proper YAML nesting.
E.G. -

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
```

**4. spec**
The spec section provides specific configuration details for the object. For a Pod, this is where you define its containers. Since a Pod can run multiple containers, the containers field is an array.
The dash (-) indicates a list item, and each container must be defined with at least name and image keys.

Below is the complete YAML configuration for our Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx-container
      image: nginx
```

Creating and Verifying the Pod :

```
kubectl create -f pod-definition.yaml
```

to check status run following command :

```
kubectl get pods
```

to get details :

```
kubectl describe pod myapp-pod
```

## B. Create a Kubernetes Pod using a YAML definition file instead of the "kubectl run" command

Create the Pod on your Kubernetes cluster using your YAML file. You can use either the kubectl create or kubectl apply command. Here's an example with kubectl apply :

```
kubectl apply -f pod-definition.yaml
```

Determine Node Placement : To see which nodes are hosting your pods, run:

```
kubectl get pods -o wide
```

Deleting the Faulty Web App Pod

```
kubectl delete pod webapp
```

GENERATING A YAML FILE :

```
kubectl run redis --image=redis123 --dry-run=client -o yaml > redis.yaml
```

---

# REPLICASET

A replication controller ensures high availability by creating and maintaining the desired number of pod replicas.
Even if you intend to run a single pod, a replication controller adds redundancy by automatically creating a replacement if the pod fails.
Beyond availability, replication controllers also help distribute load. When user demand increases, additional pods can better balance that load. If resources on a particular node become scarce, new pods can be scheduled across other nodes in your cluster.

## Creating a Replication Controller

Like any Kubernetes manifest, the file contains four main sections: apiVersion, kind, metadata, and spec.
Example yaml file :

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myapp-rc
  labels:
    app: myapp
    type: front-end
spec:
  replicas: 3
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
      - name: nginx-container
        image: nginx
```

helper cmds :

```
kubectl get replicationcontroller
kubectl get pods
```

## Introducing ReplicaSet

A ReplicaSet is a modern alternative to the replication controller, using an updated API version and some improvements. Here are the key differences:
- API Version: Use apps/v1 for a ReplicaSet.
- Selector: In addition to metadata and specification, a ReplicaSet requires a selector to explicitly determine which pods to manage. This is defined using matchLabels, which can also capture pods created before the ReplicaSet if they match the criteria.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-replicaset
  labels:
    app: myapp
    type: front-end
spec:
  replicas: 3
  selector:
    matchLabels:
      type: front-end
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
      - name: nginx-container
        image: nginx
```

helper cmds :

```
kubectl get replicaset
kubectl get pods
```

## Labels and Selectors

Labels in Kubernetes are critical because they enable controllers, such as ReplicaSets, to identify and manage the appropriate pods within a large cluster.
e.g. : assign a label (e.g., tier: front-end) to each pod. Then, use a selector to target those pods:

```yaml
selector:
  matchLabels:
    tier: front-end
```

The pod definition should similarly include the label:

```yaml
metadata:
  name: myapp-pod
  labels:
    tier: front-end
```

Even if three pods with matching labels already exist in your cluster, the template section in the ReplicaSet specification remains essential. It serves as the blueprint for creating new pods if any fail, ensuring the desired state is consistently maintained.

## Scaling the ReplicaSet

**1. Update the Definition File**
Modify the replicas value in your YAML file (e.g., change from 3 to 6) and update the ReplicaSet with:

```
kubectl replace -f replicaset-definition.yml
```

**2. Use the kubectl scale Command**

```
kubectl scale --replicas=6 -f replicaset-definition.yml
```

Keep in mind that if you scale using the kubectl scale command, the YAML file still reflects the original number of replicas. To maintain consistency, it may be necessary to update the YAML file after scaling.
for deletion use following cmd :

```
kubectl delete replicaset <replicaset-name>
```

---

# Deployment

deployments—an abstraction that simplifies managing your applications in a production environment. Rather than interacting directly with pods and ReplicaSets, deployments offer advanced features that enable you to:
- Deploy multiple instances of your application (like a web server) to ensure high availability and load balancing.
- Seamlessly perform rolling updates for Docker images so that instances update gradually, reducing downtime.
- Quickly roll back to a previous version if an upgrade fails unexpectedly.
- Pause and resume deployments, allowing you to implement coordinated changes such as scaling, version updates, or resource modifications.

A deployment, however, sits at a higher level, automatically managing ReplicaSets and pods while providing enhanced features like rolling updates and rollbacks.
Below is an example of a correct deployment definition file:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
    type: front-end
spec:
  replicas: 3
  selector:
    matchLabels:
      type: front-end
  template:
    metadata:
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
        - name: nginx-container
          image: nginx
```

NOTE : To view all the created Kubernetes objects—deployments, ReplicaSets, pods, and more—use the following command:

```
kubectl get all
```

---

# SERVICES

Kubernetes services allow different sets of Pods to interact with each other. Whether connecting the front end to back-end processes or integrating external data sources, services help to decouple microservices while maintaining reliable communication.

## Types of Kubernetes Services

Kubernetes supports several service types, each serving a unique purpose:

### NodePort

Maps a port on the node to a port on a Pod.
Remember: The NodePort service type maps a specific node port (e.g., 30008) to the target port on your Pod (e.g., 80). This provides external access while keeping internal port targeting intact.

**NodePort Service Breakdown**
With a NodePort service, there are three key ports to consider:
- Target Port: The port on the Pod where the application listens (e.g., 80).
- Port: The virtual port on the service within the cluster.
- NodePort: The external port on the Kubernetes node (by default in the range 30000–32767).

e.g. :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: NodePort
  ports:
    - targetPort: 80
      port: 80
      nodePort: 30008
```

However, this YAML definition does not link the service to any Pods. To connect the service to specific Pods, a selector is used, just as in ReplicaSets or Deployments. Consider the following Pod definition:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx-container
      image: nginx
```

Now, update the service definition to include a selector that matches these labels:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: NodePort
  ports:
    - targetPort: 80
      port: 80
      nodePort: 30008
  selector:
    app: myapp
    type: front-end
```

Save the file as service-definition.yml and create the service using:

```
kubectl create -f service-definition.yml
```

You should see a confirmation message:

```
service "myapp-service" created
```

Verify the service details with:

```
kubectl get services
```

An example output might be:

```
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
kubernetes       ClusterIP   10.96.0.1        <none>        443/TCP         16d
myapp-service    NodePort    10.106.127.123   <none>        80:30008/TCP    5m
```

Access the web service externally by pointing your browser or using curl with the node IP and NodePort:

```
curl http://192.168.1.2:30008
```

In a production environment, your application is likely spread across multiple Pods for high availability and load balancing. When Pods share matching labels, the service automatically detects and routes traffic to all endpoints. Kubernetes employs a round-robin (or random) algorithm to distribute incoming requests, serving as an integrated load balancer.

Furthermore, even if your Pods are spread across multiple nodes, Kubernetes ensures that the target port is mapped on all nodes. This means you can access your web application using the IP of any node along with the designated NodePort, providing reliable external connectivity.

---

### ClusterIP

Creates a virtual IP for internal communication between services (e.g., connecting front-end to back-end servers). Because pods receive dynamic IP addresses that can change when they are recreated, relying on these IPs for internal communication is impractical. Also there is issue in identifying which pod should handle the request.
This service provides a fixed Cluster IP or a service name, allowing other pods to access them without worrying about individual IPs. The service automatically load-balances incoming requests among the available pods.
Each service in Kubernetes is automatically assigned an IP and DNS name within the cluster. This Cluster IP should be used by other pods when accessing the service, ensuring consistent and reliable connectivity.
Below is a sample YAML configuration for creating a service named "back-end". This service exposes port 80 on the Cluster IP, forwarding requests to the back-end pods that match the specified labels (app: myapp and type: back-end). The targetPort is set to 80, matching the port where the back-end container listens:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: back-end
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: myapp
    type: back-end
```

To create the service, run the following command:

```
kubectl create -f service-definition.yml
```

After deploying the service, verify its status with:

```
kubectl get services
```

### LoadBalancer

Provisions an external load balancer (supported in cloud environments) to distribute traffic across multiple Pods. While NodePort works, it forces users to remember multiple IP-port pairs, which can be inconvenient. Keep in mind that the LoadBalancer service type only functions as intended on supported cloud environments. In unsupported settings—such as VirtualBox—the LoadBalancer type behaves like NodePort by exposing the service on a high port without providing external load balancing.

NOTE : The default Kubernetes service is automatically created and managed by the system.

---

# Namespaces

## Default Namespace and System Namespaces

By default, when you create objects such as pods, deployments, and services in your cluster, they are placed within a specific namespace (similar to being "inside a house"). The default namespace is automatically created during the Kubernetes cluster setup. Additionally, several system namespaces are created at startup:

- **kube-system:** Contains core system components like network services and DNS, segregated from user operations to prevent accidental changes.
- **kube-public:** Intended for resources that need to be publicly accessible across users.

Note: If you're running a small environment or a personal cluster for learning, you might predominantly use the default namespace. In enterprise or production environments, however, namespaces provide essential isolation and resource management by allowing environments like development and production to coexist on the same cluster.

Within a single namespace, resources can refer to each other directly via their simple names. For example, a web application pod in the default namespace can access a database service simply by using its service name:
If the web app pod needs to communicate with a service located in a different namespace, you must use its fully qualified DNS name. For example, connecting to a database service named "db-service" in the "dev" namespace follows this format:

```
mysql.connect("db-service.dev.svc.cluster.local")
```

Here, "svc" indicates the service subdomain, followed by the namespace ("dev") and the service name, ending with the default domain "cluster.local".

To list pods within the kube-system namespace:

```
kubectl get pods --namespace=kube-system
```

for default its :

```
kubectl get pods
```

Creating Pods in Specific Namespaces:

```
kubectl create -f pod-definition.yml --namespace=dev
```

yaml With Namespace Specification:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  namespace: dev
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx-container
      image: nginx
```

### Creating a Namespace

You can create a namespace using a YAML file. For example, create a file named namespace-dev.yml:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

```
kubectl create -f namespace-dev.yml
```

Alternatively, create a namespace directly through the command line:

```
kubectl create namespace dev
```

```
kubectl get pods --all-namespaces
```

To ensure that no single namespace overconsumes cluster resources, Kubernetes allows you to define ResourceQuotas. For example, create a file named compute-quota.yaml with the following content:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 5Gi
    limits.cpu: "10"
    limits.memory: 10Gi
```

Apply this configuration with:

```
kubectl create -f compute-quota.yaml
```

This configuration guarantees that the "dev" namespace does not exceed the specified resource limits.

### Conclusion

Namespaces are a fundamental component in Kubernetes, enabling you to segment and manage resources effectively. Whether you're isolating system components or separating development and production environments, using namespaces along with appropriate policies and resource quotas leads to a more efficient and organized cluster management.

---

# Imperative vs Declarative

## Imperative Approach

The imperative approach in Kubernetes involves executing specific commands to create, update, or delete objects. This method instructs Kubernetes on both what needs to be done and how it should be done. For example, you might run these commands:

```
kubectl run --image=nginx nginx
kubectl create deployment --image=nginx nginx
kubectl expose deployment nginx --port 80
kubectl edit deployment nginx
kubectl scale deployment nginx --replicas=5
kubectl set image deployment nginx nginx=nginx:1.18
kubectl create -f nginx.yaml
kubectl replace -f nginx.yaml
kubectl delete -f nginx.yaml
```

## Declarative Approach [preferred - because it helps better in tracking as file and state changes can be maintained on github]

The declarative approach enables you to specify the desired state of your infrastructure through configuration files (typically written in YAML). For example, defining a pod in a file (nginx.yaml) looks like this:

```yaml
# nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx-container
      image: nginx
```

Applying the configuration is as simple as running:

```
kubectl apply -f nginx.yaml
```

Kubernetes will create or update the object automatically to match the state described in your YAML file. When you need to update the configuration—say, to change the image version—you modify the YAML file and apply it again with:

```
kubectl apply -f nginx.yaml
```

This method ensures that your configuration files remain the single source of truth, which is especially valuable in team environments where version-controlled definitions are critical.

---

# Kubectl Apply Command

Apply configuration with:

```
kubectl apply -f nginx.yaml
```

You can also apply all configuration files within a directory:

```
kubectl apply -f /path/to/config-files
```

When you run the kubectl apply command, it compares three sources:
- The local configuration file (e.g., nginx.yaml).
- The live object configuration stored on the Kubernetes cluster.
- The last applied configuration stored as an annotation on the live object.

If the object does not exist, Kubernetes creates it based on your local configuration. During creation, Kubernetes internally adds additional fields to monitor the object's status. Notice that the YAML configuration is converted to JSON and stored as the "last applied configuration" in an annotation. This information is used during subsequent updates to identify any differences.

---

# SCHEDULING

## Manual Scheduling

assign pods to nodes without relying on Kubernetes' built-in scheduler.

### Understanding the Default Scheduler Behavior

Every pod definition includes a field called nodeName, which is left unset by default. The Kubernetes scheduler automatically scans for pods without a nodeName and selects an appropriate node by updating this field and creating a binding object.
Typically, you do not include the nodeName field in your manifest. The scheduler uses this field only after selecting a node for the pod.

To manually assign a pod to a specific node during creation, populate the nodeName field in the manifest. For example, to schedule the pod on a node called "node02", update your manifest as follows:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    name: nginx
spec:
  nodeName: node02
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 8080
```

### Reassigning a Running Pod Using a Binding Object

If a pod is already running and you need to change its node assignment, you cannot modify its nodeName directly. In this scenario, you can create a binding object that mimics the scheduler's behavior.

Create a binding object that specifies the target node ("node02"):

```yaml
apiVersion: v1
kind: Binding
metadata:
  name: nginx
target:
  apiVersion: v1
  kind: Node
  name: node02
```

The original pod definition remains unchanged:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 8080
```

Convert the YAML binding to JSON (e.g., save it as binding.json) and send a POST request to the pod's binding API using curl:

```
curl --header "Content-Type: application/json" --request POST --data @binding.json http://$SERVER/api/v1/namespaces/default/pods/nginx/binding
```

This binding instructs Kubernetes to assign the existing pod to the specified node without altering its original manifest.

---

## Labels and Selectors

labels and selectors help group and filter items effectively, and we will also discuss how annotations are used to store additional metadata.
- labels - labels attached.
- selector - select using labels.

### Specifying Labels in Kubernetes

To apply labels to a Kubernetes object such as a Pod, include a labels section under the metadata field in its definition file. Consider the following Pod definition example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    app: App1
    function: Front-end
spec:
  containers:
  - name: simple-webapp
    image: simple-webapp
    ports:
    - containerPort: 8080
```

After creating the Pod, you can retrieve it using the kubectl get pods command with a selector. For example:

```
kubectl get pods --selector app=App1
NAME            READY   STATUS      RESTARTS   AGE
simple-webapp   0/1     Completed   0          1d
```

When creating a ReplicaSet to manage three Pods, you first label the Pod definitions and then use a selector in the ReplicaSet definition to ensure the correct Pods are grouped together.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: simple-webapp
  labels:
    app: App1
    function: Front-end
spec:
  replicas: 3
  selector:
    matchLabels:
      app: App1
  template:
    metadata:
      labels:
        app: App1
        function: Front-end
    spec:
      containers:
      - name: simple-webapp
        image: simple-webapp
```

---

## Taints and Tolerations

Think of it like using a repellent on a person to keep bugs away. In Kubernetes, a taint is applied to a node (the repellent) to repel pods unless they have a matching toleration (an immunity to the repellent). This ensures that only intended pods can run on tainted nodes.

### Tainting a Node

To taint a node, use the following command where you specify the node name along with a taint in the format of a key-value pair and taint effect:

```
kubectl taint nodes node-name key=value:taint-effect
```

For example, to taint node one so that it only accepts pods associated with the application blue, run:

```
kubectl taint nodes node1 app=blue:NoSchedule
```

There are three taint effects:
- **NoSchedule:** Pods that do not tolerate the taint will not be scheduled on the node.
- **PreferNoSchedule:** The scheduler tries to avoid placing non-tolerating pods on the node, but it is not strictly enforced.
- **NoExecute:** New pods without a matching toleration will not be scheduled, and existing pods will be evicted from the node.

### Adding Tolerations to Pods

To allow a specific pod to run on a tainted node, add a toleration to the pod's manifest. For example, to enable Pod D to run on node one (tainted with "app=blue:NoSchedule"), modify the pod definition as follows:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: nginx-container
      image: nginx
  tolerations:
    - key: "app"
      operator: "Equal"
      value: "blue"
      effect: "NoSchedule"
```

### Master Node Taints

By default, Kubernetes applies a taint to master nodes to prevent workload pods from being scheduled there. This setup ensures that master nodes are reserved for critical system components. To check the taint on a master node, use the following command:

```
kubectl describe node kubemaster | grep Taint
```

The output will display a taint similar to:

```
Taints:             node-role.kubernetes.io/master:NoSchedule
```

This configuration follows best practices by ensuring that only essential components run on the master node.

---

## Node Affinity

Node affinity overcomes these limitations by allowing more expressive rules. The example below demonstrates how to schedule a pod on a node labeled as large using node affinity:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: data-processor
      image: data-processor
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: size
                operator: In
                values:
                  - Large
```

- The affinity key under spec introduces the nodeAffinity configuration.
- The field requiredDuringSchedulingIgnoredDuringExecution indicates that the scheduler must place the pod on a node meeting the criteria. Once the pod is running, any changes to node labels are ignored.
- The nodeSelectorTerms array contains one or more matchExpressions. Each expression specifies a label key, an operator, and a list of values. Here, the In operator ensures that the pod is scheduled only on nodes where the label size includes 'Large'.

To allow for more flexible scheduling, such as permitting a pod to run on either large or medium nodes, simply add additional values to the list. Alternatively, you can use the NotIn operator to explicitly avoid placing a pod on nodes with specific labels. For example, to avoid nodes labeled as small:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: data-processor
      image: data-processor
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: size
                operator: NotIn
                values:
                  - Small
```

In cases where you only need to verify the presence of a label without checking for specific values, the Exists operator is useful. When using Exists, you do not provide a list of values:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: data-processor
    image: data-processor
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: Exists
```

### Scheduling Behavior

Once a pod is scheduled using node affinity rules, these rules are only evaluated during scheduling. Changes to node labels after scheduling will not affect a running pod due to the "ignored during execution" behavior.

There are two primary scheduling behaviors for node affinity:

**Required During Scheduling, Ignored During Execution**
- The pod is scheduled only on nodes that fully satisfy the affinity rules.
- Once running, changes to node labels do not impact the pod.

**Preferred During Scheduling, Ignored During Execution**
- The scheduler prefers nodes that meet the affinity rules but will place the pod on another node if no matching nodes are available.

In Kubernetes, taints and tolerations are primarily used to repel pods from nodes unless they explicitly tolerate the taint, whereas node affinity is used to attract pods to nodes that satisfy specific label criteria.

---

## Resource Requests

Resource requests define the minimum CPU and memory a container is guaranteed when scheduled. Specifying a request, for instance, of 1 CPU and 1 GB memory in a pod definition ensures that the pod will only be scheduled on a node that can provide these minimum resources. The Kubernetes scheduler searches for a node that meets these requirements, guaranteeing the pod access to the declared resources.

Below is an example snippet of a pod definition with resource requests set to 4 Gi of memory and 2 CPU cores:

### Resource Limits

Setting a limit, such as 1 vCPU, prevents the container from using more than that allocation. Similarly, memory limits restrict its maximum memory usage. Note that while CPU usage is throttled when a pod reaches its limit, memory usage is not. If a pod exceeds its memory limit, it may be terminated due to an Out Of Memory (OOM) error.

```yaml
piVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
  labels:
    name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "1Gi"
        cpu: 1
      limits:
        memory: "2Gi"
        cpu: 2
```

### Limit Ranges

For example, you can create a LimitRange to enforce CPU constraints:

```yaml
# limit-range-cpu.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-resource-constraint
spec:
  limits:
    - default:
        cpu: 500m
      defaultRequest:
        cpu: 500m
      max:
        cpu: "1"
      min:
        cpu: 100m
      type: Container
```

Likewise, to set memory constraints, use the following configuration:

```yaml
# limit-range-memory.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: memory-resource-constraint
spec:
  limits:
    - default:
        memory: 1Gi
      defaultRequest:
        memory: 1Gi
      max:
        memory: 1Gi
      min:
        memory: 500Mi
      type: Container
```

---

## DaemonSets

DaemonSets ensure that exactly one copy of a pod runs on every node in your Kubernetes cluster.
When you add a new node, the DaemonSet automatically deploys the pod on the new node. Likewise, when a node is removed, the corresponding pod is also removed. This guarantees that a single instance of the pod is consistently available on each node.

### Use Cases for DaemonSets

DaemonSets are particularly useful in scenarios where you need to run background services or agents on every node. Some common use cases include:
- Monitoring agents and log collectors: Deploy monitoring tools or log collectors across every node to ensure comprehensive cluster-wide visibility without manual intervention.
- Essential Kubernetes components: Deploy critical components, such as kube-proxy, which Kubernetes requires on all worker nodes.
- Networking solutions: Ensure consistent deployment of networking agents like those used in VNet or weave-net across all nodes.

### Creating a DaemonSet

Creating a DaemonSet is analogous to creating a ReplicaSet. The DaemonSet YAML configuration consists of a pod template under the template section and a selector that binds the DaemonSet to its pods.

```yaml
# daemon-set-definition.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
spec:
  selector:
    matchLabels:
      app: monitoring-agent
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      containers:
        - name: monitoring-agent
          image: monitoring-agent
```

---

advance concepts : static pods , priority classes , multiple schedular , admission controllers

---

# LOGGING AND MONITORING

For nodes, consider monitoring the following:
- Total number of nodes in the cluster
- Health status of each node
- Performance metrics such as CPU, memory, network, and disk utilization

For pods, focus on:
- The number of running pods
- CPU and memory consumption for every pod

Because Kubernetes does not include a comprehensive built-in monitoring solution, you must implement an external tool. Popular open-source monitoring solutions include Metrics Server, Prometheus, and Elastic Stack.
Historically, Heapster provided monitoring and analysis support for Kubernetes. Its streamlined successor, Metrics Server, is now the standard for monitoring Kubernetes clusters.

Metrics Server is designed to be deployed once per Kubernetes cluster. It collects metrics from nodes and pods, aggregates the data, and retains it in memory. Keep in mind that because Metrics Server stores data only in memory, it does not support historical performance data. For long-term metrics, consider integrating more advanced monitoring solutions.
deploy Metrics Server by cloning the GitHub repository and applying its deployment files:

```
git clone https://github.com/kubernetes-incubator/metrics-server.git
kubectl create -f deploy/1.8+/
```

## Managing Application Logs

**Logging in Docker :** Docker containers typically log events to the standard output.

**Logging in Kubernetes :** Deploying the same Docker image within a Kubernetes pod leverages Kubernetes' logging capabilities. To get started, create a pod using the following YAML definition:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: event-simulator-pod
spec:
  containers:
    - name: event-simulator
      image: kodekloud/event-simulator
```

Create the pod with this command:

```
kubectl create -f event-simulator.yaml
```

Once the pod is running, view the live logs using:

```
kubectl logs -f event-simulator-pod
```

---

# Application Lifecycle Management

## Rolling Updates and Rollbacks

When you create a deployment, Kubernetes initiates a rollout that establishes the first deployment revision (revision one). Later, when you update your application—say by changing the container image version—Kubernetes triggers another rollout, creating a new revision (revision two). These revisions help you track changes and enable rollbacks to previous versions if issues arise.
To monitor and review these rollouts, you can use the following commands:

Check the rollout status:

```
kubectl rollout status deployment/myapp-deployment
```

View the history of rollouts:

```
kubectl rollout history deployment/myapp-deployment
```

## Deployment Strategies

- One approach is the "recreate" strategy, which involves shutting down all existing instances before deploying new ones. However, this method results in temporary downtime as the application becomes inaccessible during the update.
- A more seamless approach is the "rolling update" strategy. Here, instances are updated one at a time, ensuring continuous application availability throughout the process. If no strategy is specified when creating a deployment, Kubernetes uses the rolling update strategy by default.

**Updating a Deployment :** A common practice is to update your deployment definition file and then apply the changes.
During an upgrade, Kubernetes creates a new ReplicaSet for the updated containers while the original ReplicaSet continues to run the old version. This rolling update process ensures that new pods replace the old ones gradually without causing downtime.

Check the status of the rollout:

```
kubectl rollout status deployment/myapp-deployment
```

If an issue is detected after an upgrade, you can revert to the previous version using the rollback feature. To perform a rollback, run:

```
kubectl rollout undo deployment/myapp-deployment
```

## Commands and Arguments in Kubernetes

Previously, we built a simple Docker image—named "ubuntu-sleeper"—that executes a sleep command for a specified number of seconds. By default, running a container with this image makes it sleep for five seconds. However, you can easily change this behavior by passing a command-line argument.

**Overriding Default Behavior with Arguments:**
Suppose we want to create a pod using the "ubuntu-sleeper" image. We begin with a basic pod definition where the pod's name and image are specified. When the pod is created, it starts a container that runs the default sleep command (sleeping for five seconds) and then exits. To modify the sleep duration to 10 seconds, simply append an additional argument in the pod specification. Any argument provided in the Docker run command correlates with the args property in the pod definition file (formatted as an array).

docker command :

```
docker run --name ubuntu-sleeper ubuntu-sleeper 10
```

Pod definition YAML:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    args: ["10"]
```

Docker command example with overridden ENTRYPOINT:

```
docker run --name ubuntu-sleeper \
  --entrypoint sleep2.0 \
  ubuntu-sleeper 10
```

Pod definition YAML with overridden ENTRYPOINT:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
    - name: ubuntu-sleeper
      image: ubuntu-sleeper
      command: ["sleep2.0"]
      args: ["10"]
```

Remember that specifying the command in a pod definition replaces the Dockerfile's ENTRYPOINT entirely, while the args field only overrides the default parameters defined by CMD.

## Configure Environment Variables in Applications

Below is an example pod definition that explicitly sets the environment variable:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 8080
      env:
        - name: APP_COLOR
          value: pink
```

In this configuration, the container running the simple-webapp-color image will have APP_COLOR set to "pink".

To reference a ConfigMap for the environment variable, update the definition as follows:

```yaml
env:
  - name: APP_COLOR
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: color
```

Similarly, to source the environment variable from a Secret, configure it like this:

```yaml
env:
  - name: APP_COLOR
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: color
```

## Centralizing Configuration with ConfigMaps

To simplify the management of environment configurations, you can externalize the data using a ConfigMap. With this approach, Kubernetes injects centrally stored key–value pairs into your pods during creation.

### Creating ConfigMaps

**a. Imperative Approach**

```
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MOD=prod
```

You can also generate a ConfigMap from a file using the --from-file option. For instance, if you have a file named app_config.properties containing configuration data:

```
kubectl create configmap app-config --from-file=app_config.properties
```

**b. Declarative Approach**
With a declarative approach, you define your ConfigMap in a YAML file and apply it with kubectl. Here is an example ConfigMap definition:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: prod
```

Create the ConfigMap in your cluster with:

```
kubectl create -f config-map.yaml
```

For larger deployments, consider organizing multiple ConfigMaps by logical grouping, such as for your application, MySQL, and Redis.

### Using ConfigMaps

After creating a ConfigMap, configure your pod to use the configuration data. Below is an example pod definition that injects the app-config ConfigMap into the container as environment variables:

```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
  labels:
    name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 8080
  envFrom:
    - configMapRef:
        name: app-config
```

## Secrets

Welcome to this comprehensive guide on managing Secrets in Kubernetes. In this article, we explain how to securely handle sensitive data (such as passwords and keys) in your Kubernetes deployments while avoiding common pitfalls like hardcoding credentials in your application.

Below is an illustration of Secret data in their encoded and decoded forms:

**Encoded Values**
```
DB_Host: bXlzcWw=
DB_User: cm9vdA==
DB_Password: cGFzd3Jk
```

**Decoded Values**
```
DB Host: mysql
DB User: root
DB Password: paswrd
```

### Imperative Creation of a Secret

```
kubectl create secret generic app-secret \
  --from-literal=DB_Host=mysql \
  --from-literal=DB_User=root \
  --from-literal=DB_Password=paswd
```

Alternatively, create a Secret from a file with the --from-file option:

```
kubectl create secret generic app-secret --from-file=app_secret.properties
```

### Declarative Creation of a Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_Host: bXlzcWw=
  DB_User: cm9vdA==
  DB_Password: cGFzd3Jk
```

Apply the definition with the following command:

```
kubectl create -f secret-data.yaml
```

### Viewing and Decoding Secrets

After creating a Secret, you can list and inspect it with the following commands:

List Secrets:

```
kubectl get secrets
```

Expected output:

```
NAME          TYPE    DATA   AGE
app-secret    Opaque    3    10m
```

Describe a Secret (without showing sensitive data):

```
kubectl describe secret app-secret
```

View the encoded data in YAML format:

```
kubectl get secret app-secret -o yaml
```

If you need to decode an encoded value, use the base64 --decode command:

```
echo -n 'bXlzcWw=' | base64 --decode
echo -n 'cm9vdA==' | base64 --decode
echo -n 'cGFzd3Jk' | base64 --decode
# Output: paswrd
```

### Injecting as Environment Variables

Below is an example Pod definition that injects the Secret as environment variables:

```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
  labels:
    name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    ports:
    - containerPort: 8080
    envFrom:
    - secretRef:
        name: app-secret
```

### Mounting Secrets as Files

Alternatively, mount the Secret as files within a volume. Each key in the Secret becomes a separate file:

```yaml
volumes:
- name: app-secret-volume
  secret:
    secretName: app-secret
```

After mounting, listing the directory contents should display each key as a file:

```
ls /opt/app-secret-volumes
# Output: DB_Host  DB_Password  DB_User
```

To view the content of a specific file, such as the DB password:

```
cat /opt/app-secret-volumes/DB_Password
# Output: paswrd
```

[NOTE : Secrets offer only Base64 encoding. For enhanced security, consider enabling encryption at rest for etcd.
Limit access to Secrets using Role-Based Access Control (RBAC). Restrict permissions to only those who require it.
For even greater security, explore third-party secret management solutions such as AWS Secrets Manager, Azure Key Vault, GCP Secret Manager, or Vault.]

## Multi Container Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    name: simple-webapp
spec:
  containers:
    - name: simple-webapp
      image: simple-webapp
      ports:
        - containerPort: 8080
    - name: log-agent
      image: log-agent
```

This configuration ensures that both containers share the same lifecycle, network, and storage resources, allowing them to work together seamlessly.

## Scaling in Kubernetes

For the cluster infrastructure:
- Horizontal scaling: Add more nodes to the cluster.
- Vertical scaling: Increase resources (CPU, memory) on existing nodes.

For workloads:
- Horizontal scaling: Create more Pods.
- Vertical scaling: Increase resource limits and requests for existing Pods.

how to scale : Manual scaling and automated scaling both have their place. Manual scaling involves direct intervention and command execution, while automated scaling leverages Kubernetes controllers for dynamic adjustments.

---

# SECURITY

explain how access is granted to a Kubernetes cluster, how various actions are controlled, and review the different authentication mechanisms available. Additionally, we will examine the default configurations of a cluster and demonstrate how to view the configurations of an existing cluster. We then discuss TLS certificates and the role they play in securing components within the cluster.

## Securing Cluster Hosts

The security of your Kubernetes cluster begins with the hosts themselves. Protect your underlying infrastructure by following these practices:
- Disable root access.
- Turn off password-based authentication.
- Enforce SSH key-based authentication.
- Implement additional measures to secure your physical or virtual systems.

A compromised host can expose your entire cluster, so securing these systems is a critical first step.

## API Server Access Control

### Authentication

Authentication verifies the identity of a user or service before granting access to the API server. Kubernetes offers various authentication mechanisms to suit different security needs:
- Static user IDs and passwords
- Tokens
- Client certificates
- Integration with external authentication providers (e.g., LDAP)

Additionally, service accounts support non-human processes. Detailed guidance on these methods is available in dedicated sections of our documentation.

### Authorization

After authentication, authorization determines what actions a user or service is allowed to perform. The default mechanism, Role-Based Access Control (RBAC), associates identities with specific permissions. Kubernetes also supports:
- Attribute-Based Access Control (ABAC)
- Node Authorization
- Webhook-based authorization

These mechanisms enforce granular access control policies, ensuring that authenticated entities can perform only the operations they are permitted to execute.

## Securing Component Communications

Secure communications between Kubernetes components are enabled via TLS encryption. This ensures that data transmitted between key components remains confidential and tamper-proof.

## Network Policies

By default, pods in a Kubernetes cluster communicate freely with one another. To restrict unwanted interactions and enhance security, Kubernetes provides network policies. These policies allow you to:
- Control traffic flow between specific pods
- Enforce security rules at the network level

## Authentication

- Human Users: Administrators and developers who perform administrative tasks and deploy applications.
- Service Accounts: Robot users that represent processes, services, or third-party applications.

Kubernetes does not manage human user accounts natively. Instead, it relies on external systems such as static files containing user credentials, certificates, or third-party identity services (e.g., LDAP, Kerberos) for authentication. Every request—whether from the kubectl command-line tool or direct API calls—is processed by the Kubernetes API server, which authenticates the incoming requests using one or more available authentication mechanisms.
Hence this has to be handled by IAM in GCP.

## SSL/TLS certificates

A TLS certificate establishes trust during a transaction by ensuring that communications are encrypted and that the server is indeed who it claims to be.
Asymmetric encryption uses a pair of keys: a private key and a public key. You can think of these as a private key and a public lock. The private key remains securely with the owner, while the public lock can be shared openly. Data encrypted with the public key can only be decrypted using its corresponding private key, ensuring that intercepted data remains secure.
By using key pairs, you can generate a private key (id_rsa) and a public key (id_rsa.pub). The private key stays secure on your device, while the public key is added to the server's SSH authorized keys file. When you initiate an SSH connection, you specify your private key to authenticate.
If you have multiple servers, simply copy your public key to each server's authorized keys file so you can authenticate using the same private key on all servers.

Here's how the process works for a web server using HTTPS:
1. The server generates a key pair (private and public keys).
2. Upon a user's initial HTTPS request, the server sends its public key embedded within a certificate.
3. The client's browser encrypts a newly generated symmetric key using the server's public key.
4. The encrypted symmetric key is sent back to the server.
5. The server decrypts the symmetric key using its private key.
6. All subsequent communications are encrypted with this symmetric key.

A certificate contains essential details that help verify its authenticity:
- Identity of the issuing authority
- The server's public key
- Domain and other related information

Browsers rely on Certificate Authorities (CAs) to sign and validate certificates. Renowned CAs, such as Symantec, DigiCert, Komodo, and GlobalSign, use their private keys to sign certificate signing requests (CSRs). When you generate a CSR for your web server, it is sent to a CA for signing:

Once your details are validated, the CA signs the certificate and sends it back to be installed on your web server. When a user accesses your website, the process is as follows:
1. The server presents the certificate.
2. The browser validates it using pre-installed CA public keys.
3. Upon successful validation, the browser and server establish a secure session using a symmetric key exchanged via asymmetric encryption.

**Naming Conventions :** Certificate files follow specific naming conventions. Public key certificates typically have a .crt or .pem extension (e.g., server.crt, server.pem, client.crt, client.pem). In contrast, private keys usually include the word "key" in their file name or extension (e.g., server.key or server-key.pem). If a file name lacks "key," it is almost certainly a public key certificate.

A Kubernetes cluster consists of master and worker nodes that require secure, encrypted communication. Whether the connection is being made by an administrator using the kubectl utility or directly interacting with the Kubernetes API, a secure TLS connection is essential. Additionally, services within the cluster use server certificates to secure their communications, while client certificates authenticate users or other cluster components.

## Role Based Access Controls

### Creating a Role

To define a role, create a YAML file that sets the API version to rbac.authorization.k8s.io/v1 and the kind to Role. In this example, we create a role named developer to grant developers specific permissions. The role includes a list of rules where each rule specifies the API groups, resources, and allowed verbs. For resources in the core API group, provide an empty string ("") for the apiGroups field.

For instance, the following YAML definition grants developers permissions on pods (with various actions) and allows them to create ConfigMaps:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
```

Create the role by running:

```
kubectl create -f developer-role.yaml
```

### Creating a Role Binding

After defining a role, you need to bind it to a user. A role binding links a user to a role within a specific namespace. In this example, we create a role binding named devuser-developer-binding that grants the user dev-user the developer role.

Below is the combined YAML definition for both creating the role and its corresponding binding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

Create the role binding using the command:

```
kubectl create -f devuser-developer-binding.yaml
```

### Verifying Roles and Role Bindings

After applying your configurations, it's important to verify that the roles and role bindings have been created correctly.

To list all roles in the current namespace, execute:

```
kubectl get roles
```

Example output:

```
NAME        AGE
developer   4s
```

```
kubectl get rolebindings
```

### Testing Permissions with kubectl auth

You can test whether you have the necessary permissions to perform specific actions by using the kubectl auth can-i command. For example, to check if you can create deployments, run:

```
kubectl auth can-i create deployments
```

---

In our previous discussion, we covered roles and role bindings, which are namespace-specific. In this article, we extend that concept by introducing cluster roles and cluster role bindings, allowing you to manage permissions across your entire Kubernetes cluster.

Below is an example of a cluster role definition file named cluster-admin-role.yaml. This YAML file defines a ClusterRole that grants administrative permissions on nodes:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "get", "create", "delete"]
```

Once the ClusterRole is created, you bind it to a user through a ClusterRoleBinding object. The following example binds the cluster-administrator ClusterRole to a user named cluster-admin:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```

## Service Accounts (at Kubernetes level)

```
kubectl create serviceaccount dashboard-sa
kubectl get serviceaccount
kubectl describe serviceaccount dashboard-sa
```

To examine the token itself, view the corresponding Secret:

```
kubectl describe secret dashboard-sa-token-kbbdm
```

Sample output:

```
Name:                dashboard-sa-token-kbbdm
Namespace:           default
Labels:              <none>
Type:                kubernetes.io/service-account-token
Data
  token: eyJhbGciOiJSUzI1NiIsImtpZCI6Ij...  (truncated for privacy)
```

This token serves as the authentication bearer token for accessing the Kubernetes API. For example, using curl:

```
curl https://192.168.56.70:6443/api -k \
--header "Authorization: Bearer eyJhbgG…"
```

When deploying third-party applications (such as a custom dashboard or Prometheus) on a Kubernetes cluster, you can have Kubernetes automatically mount the service account token as a volume into the Pod. This token is typically available at the path: `/var/run/secrets/kubernetes.io/serviceaccount`.

by default every namespace includes a default service account that is automatically injected into Pods.

```
kubectl describe pod my-kubernetes-dashboard
```

### Using a Different Service Account

By default, Pods use the default service account. To assign a different service account—like the previously created dashboard-sa—update your Pod definition to include the serviceAccountName field:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  serviceAccountName: dashboard-sa
  containers:
    - name: my-kubernetes-dashboard
      image: my-kubernetes-dashboard
```

If you wish to disable the automatic mounting of the service account token, set automountServiceAccountToken to false in the Pod specification:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  automountServiceAccountToken: false
  containers:
  - name: my-kubernetes-dashboard
    image: my-kubernetes-dashboard
```

Prior to Kubernetes v1.22, service account tokens were automatically mounted from Secrets without an expiration date. Starting with v1.22, the TokenRequest API (KEP-1205) was introduced to generate tokens that are audience-bound, time-bound, and object-bound—enhancing security significantly.

## Security Contexts

In Kubernetes, containers are always encapsulated in pods. You can define security settings either at the pod level, which affects all containers in the pod, or at the container level where the settings apply specifically to one container. Note that if the same security configuration is set at both the pod and container levels, the container-specific settings take precedence over the pod-level configurations.

Below is an example of a pod definition that configures the security context at the container level, using an Ubuntu image that runs the sleep command:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1000
        capabilities:
          add: ["MAC_ADMIN"]
```

This configuration instructs Kubernetes to run the container as user ID 1000 and adds the MAC_ADMIN capability. If your goal is to enforce these security settings across all containers within a single pod, you can define the security context at the pod level instead.

## Network Policies

Consider a simple application configuration consisting of a web server, an API server, and a database server. The traffic flow is as follows:
To support this traffic flow, the following rules must be established:
- An ingress rule for the web server to allow HTTP traffic on port 80.
- An egress rule for the web server to permit traffic to port 5000 on the API server.
- An ingress rule for the API server to accept traffic on port 5000.
- An egress rule for the API server to allow traffic to port 3306 on the database server.
- An ingress rule for the database server to allow traffic on port 3306.

In a Kubernetes cluster, nodes host pods and services, each assigned a unique IP address. A crucial capability of Kubernetes is that pods can communicate with one another without extra configuration—such as setting up custom routes. Typically, all pods reside in a virtual private network (VPN) that spans the entire cluster, allowing them to interact using pod IPs, pod names, or configured services.

By default, Kubernetes employs an "all-allow" rule permitting any pod to communicate with every other pod or service within the cluster. but in example that we took we want to control the traffic.

### Implementing a Network Policy

To apply a network policy, you assign labels to pods and define matching selectors in the network policy object.

Below is the complete network policy configuration:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
    ports:
    - protocol: TCP
      port: 3306
```

In this configuration:
- The podSelector targets the database pod through its label.
- The policyTypes field specifies that only ingress traffic is affected.
- The ingress rule allows traffic specifically from pods with the label name: api-pod on TCP port 3306.

Keep in mind that isolation only applies to the traffic explicitly defined under policyTypes. Unspecified traffic is automatically allowed by default.

```
kubectl create -f db-policy.yaml
```

### Blocking All Ingress Traffic to the Database Pod

The following YAML file defines a policy that denies all ingress traffic by default since no specific ingress rules are provided:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
```

ie. If no ingress rules are specified, Kubernetes treats the policy as a complete block for incoming traffic.

### Allowing Ingress Traffic from the API Pod

The next step is to allow ingress traffic from the API pod on port 3306. Because responses to permitted traffic are automatically allowed, configuring an ingress rule is sufficient. In this rule, the source is defined by pod selectors (and optionally namespace selectors) while the destination port is specified.

For instance, to allow traffic only from pods labeled name: api-pod within namespaces labeled prod—and also permit traffic from a backup server with IP 192.168.5.10—update the network policy as follows:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
      namespaceSelector:
        matchLabels:
          name: prod
    - ipBlock:
        cidr: 192.168.5.10/32
  ports:
  - protocol: TCP
    port: 3306
```

In this configuration, the first entry under the from section restricts traffic to API pods in the production namespace (an AND condition). The second entry allows an external backup server by specifying its IP block.

Tip: Combining pod selectors with namespace selectors ensures that the rule applies only to the intended pods within the correct namespace.

### Configuring Egress Traffic

In scenarios where the database pod must initiate outbound connections (for example, sending backups to an external server), an egress rule becomes necessary. To support both ingress and egress traffic, include Egress in the policyTypes and specify an egress rule.

The revised policy below permits the database pod to send traffic to a backup server at IP 192.168.5.10 on port 80 while still restricting all other outbound connections:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
      namespaceSelector:
        matchLabels:
          name: prod
  ports:
  - protocol: TCP
    port: 3306
  egress:
  - to:
    - ipBlock:
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 80
```

Ensure that your egress rules cover all required outbound connections. Missing an egress rule may inadvertently block critical communication between your services.

---

# Storage in Docker

When Docker is installed, it creates a folder structure at /var/lib/docker containing subdirectories such as overlay2, containers, images, and volumes. These directories store Docker images, container runtime data, and volumes. For instance, files associated with running containers reside in the containers folder, image files are stored under images, and any created volumes are kept in the volumes folder.

If you log into the container and modify a file (say, creating temp.txt), Docker employs a copy-on-write mechanism. Before modifying a file originating from the read-only image layer, Docker first copies it to the writable layer, and subsequent changes are applied to the copied file—leaving the original image intact. When the container is removed, the writable layer and any changes in it are deleted.

## Volume Mounts

Volumes are managed by Docker and stored under /var/lib/docker/volumes. Create and mount a volume with the following commands:

```
docker volume create data_volume
docker run -v data_volume:/var/lib/mysql mysql
```

If you run a container with a volume name that doesn't exist, Docker will automatically create it:

```
docker run -v data_volume2:/var/lib/mysql MySQL
```

## Bind Mounts

Bind mounts allow you to use a specific directory from the Docker host. For example, to use data from /data/mysql, run:

```
docker run -v /data/mysql:/var/lib/mysql mysql
```

### Using the --mount Option

The --mount flag provides a more explicit syntax by requiring all parameters to be specified. The following command is equivalent to the bind mount example above:

```
docker run \
  --mount type=bind,source=/data/mysql,target=/var/lib/mysql \
  MySQL
```

## Volume Driver Plugins in Docker

By default, Docker uses the local volume driver plugin. This plugin creates a volume on the Docker host and stores its data under the /var/lib/docker/volumes directory. However, many other volume driver plugins are available, enabling you to create volumes on third-party storage solutions such as Google Compute Persistent Disks.

some volume drivers support multiple storage providers. The Rex Ray storage driver, for instance, can provision storage on AWS EBS, S3, EMC storage arrays (like Isilon and ScaleIO), Google Persistent Disk, or OpenStack Cinder. When running a Docker container, you can select a specific volume driver such as Rex Ray EBS to provision a volume from Amazon EBS. This approach ensures that your data remains safe in the cloud even if the container exits.

Below is an example of running a Docker container using a specific volume driver plugin:

```
docker run -it \
  --name mysql \
  --volume-driver rexray/ebs \
  --mount src=ebs-vol,target=/var/lib/mysql \
  MySQL
```

## How CSI Works

When a pod is created that requires persistent storage, the container orchestrator (e.g., Kubernetes) calls the "create volume" Remote Procedure Call (RPC) defined by the CSI standard. This call includes vital details such as the volume name and other parameters. The storage driver then processes the request by provisioning a new volume on the associated storage array and returns the result to the orchestrator.

Similarly, when a volume is no longer needed, the orchestrator issues a "delete volume" RPC. The storage driver responds by removing the specified volume from the storage system. The CSI specification outlines the required parameters, expected responses, and error codes for these operations, ensuring consistency and interoperability across different storage solutions.

## Volumes in Kubernetes

consider an example : the volume is configured to use the host's /data directory. This implies that any files created within the volume (for instance, the random number written to /opt/number.out in the container) are stored in /data on the node. The volume is then mounted to the /opt directory inside the container.

Below is the YAML configuration for the pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
spec:
  containers:
  - image: alpine
    name: alpine
    command: ["/bin/sh", "-c"]
    args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
    volumeMounts:
    - mountPath: /opt
      name: data-volume
  volumes:
  - name: data-volume
    hostPath:
      path: /data
      type: Directory
```

When the pod executes the command, the random number is written to /opt/number.out inside the container. Since /opt is mounted to the host's /data directory, the file persists on the host even after the pod is deleted.

## Volume Storage Options

In the above example, we utilized a hostPath volume, which is suitable for a single-node cluster. However, this approach is not ideal for multi-node clusters where each node's /data directory may differ. For such environments, Kubernetes supports several external and replicated storage solutions. Some popular storage options include: Public cloud storage solutions like AWS EBS, Azure Disk or File, and Google Persistent Disk.
For example, to use an AWS Elastic Block Store (EBS) volume, replace the hostPath configuration with AWS EBS-specific parameters such as the volume ID and filesystem type. This allows your Kubernetes cluster to leverage AWS EBS for robust, scalable storage.

| Storage Option | Use Case | Volume Type |
|---|---|---|
| Google Persistent Disk | Multi-node clusters on Google Cloud | gcePersistentDisk |

## PERSISTENT VOLUME

A persistent volume is a cluster-wide storage resource defined and managed by an administrator. Applications running on the cluster utilize these PVs by binding to them via persistent volume claims.

In environments where many users deploy multiple pods, duplicating storage configuration in every pod file can lead to redundancy and increased maintenance efforts. Any required change would need to be propagated across all pod definitions.

To solve this issue, administrators can create a centralized pool of storage. Users then request portions of this storage as needed by creating persistent volume claims (PVCs). This concept is enabled by persistent volumes (PVs).

### Creating a Persistent Volume

In this section, we will create a persistent volume using a base template. First, update the API version, set the kind to PersistentVolume, and give it a name (for example, "pv-vol1"). Under the spec section, it's necessary to define the access modes. The access modes determine how a volume can be mounted on nodes, such as:
- ReadWriteOnce: The volume can be mounted as read-write by a single node.
- ReadOnlyMany: The volume can be mounted as read-only by multiple nodes.
- ReadWriteMany: The volume can be mounted as read-write by multiple nodes.

Next, specify the storage capacity for this persistent volume. In our example, we set the capacity to 1Gi. After defining the capacity, choose the volume type. Here, we use the hostPath option, which leverages storage from the node's local directory.

Note: The hostPath option is primarily for testing or single-node setups and is not recommended for production environments.

The complete persistent volume manifest appears as follows:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /tmp/data
```

To create the persistent volume, execute the following command:

```
kubectl create -f pv-d
```

After creating the volume, you can verify its status by running:

```
kubectl get persistentvolume
```

In a production setup, replace the hostPath option with a supported storage solution, such as AWS Elastic Block Store, to ensure data durability and scalability.

## Persistent Volume Claims

Persistent volumes and persistent volume claims are two distinct objects in Kubernetes. An administrator is responsible for creating PVs, while users create PVCs to request storage resources. When a PVC is created, Kubernetes automatically binds it to a PV that meets the requested capacity, access modes, volume modes, and storage class.
Kubernetes evaluates several factors when binding a PVC to a PV. If multiple PVs can satisfy a claim, you can use labels and selectors to bind the claim to a specific volume.
It is important to note that if a smaller PVC is matched with a larger PV that meets all criteria, the unrequested capacity remains unused by any other PVC. If no PV satisfies the claim's requirements, the PVC will remain in a pending state until a new, suitable PV becomes available.

### Creating a Persistent Volume Claim

Below is an example YAML template for creating a PVC. In this configuration, we set the API version to v1 with kind PersistentVolumeClaim, and name it "myclaim". Under the specification section, the access mode is set to ReadWriteOnce, and 500 MiB of storage is requested.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

To create the PVC, save the above YAML to a file, for example, pvc-definition.yaml.

Run the command below in your terminal:

```
kubectl create -f pvc-definition.yaml
```

You can verify the created PVC by executing:

```
kubectl get persistentvolumeclaim
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES
myclaim   Pending
```

Kubernetes will inspect the available PV. Suppose, in our example, a PV is configured with 1GiB storage and compatible access modes — if it meets the PVC's criteria, it will automatically bind to the PVC. Here is an example of such a PV definition:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-voll
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  awsElasticBlockStore:
    volumeID: <volume-id>
    fsType: ext4
```

After the binding process, running the kubectl get persistentvolumeclaim command will show that the PVC has been successfully bound to the matching PV.

### Deleting a PVC and Persistent Volume Reclaim Policies

To delete a PVC, use the following command:

```
kubectl delete persistentvolumeclaim myclaim
```

When a PVC is deleted, what happens next depends on the underlying persistent volume's reclaim policy. The reclaim policy determines the fate of the PV and can be configured as follows:

| Reclaim Policy | Description |
|---|---|
| Retain | The PV remains in the cluster after the PVC is deleted. An administrator must manually reclaim it. |
| Delete | The PV is automatically deleted along with the PVC, releasing the storage on the physical device. |
| Recycle | The PV data is scrubbed before reuse by new claims. |

Note: The "Recycle" reclaim policy is deprecated in recent Kubernetes versions and might not be available in your cluster.

For example, to set the reclaim policy to Retain, you would include:

```yaml
persistentVolumeReclaimPolicy: Retain
```

Choose the reclaim policy that best fits your storage management strategy.

## Storage Class

In this lesson, we explore storage classes in Kubernetes and demonstrate how they simplify the process of storage provisioning for applications. Traditionally, administrators manually created PersistentVolumes (PVs) and PersistentVolumeClaims (PVCs) and mounted them to pods. This guide covers both static provisioning (manually creating disks and PVs) and dynamic provisioning using storage classes, making your Kubernetes storage management more efficient.

### Static Provisioning

With static provisioning, you manually create the underlying storage (for example, a Google Cloud persistent disk) and then construct a PV that references that disk. Each time an application requires storage, you must provision a disk on Google Cloud and create the corresponding PV definition.

For example, to create a persistent disk on Google Cloud, you can use the following command:

```
gcloud beta compute disks create \
  --size 1GB \
  --region us-east1 \
  pd-disk
```

Then, define your Kubernetes resources as follows:

```yaml
# pv-definition.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 500Mi
  gcePersistentDisk:
    pdName: pd-disk
    fsType: ext4
```

```yaml
# pvc-definition.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
spec:
  containers:
    - image: alpine
      name: alpine
      command: ["/bin/sh", "-c"]
      args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
      volumeMounts:
        - mountPath: /opt
          name: data-volume
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: myclaim
```

In this setup, the PVC binds to the manually created PV that refers to your existing Google Cloud persistent disk.

### Dynamic Provisioning with Storage Classes

Dynamic provisioning removes the need for manual storage pre-provisioning. When you create a PVC, the associated storage class automatically provisions the necessary PV using the defined provisioner.

First, create a storage class object that specifies the provisioner (in this case, Google Cloud's persistent disk):

```yaml
# sc-definition.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: google-storage
provisioner: kubernetes.io/gce-pd
```

With the storage class in place, update your PVC to reference it for dynamic provisioning:

```yaml
# pvc-definition.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: google-storage
  resources:
    requests:
      storage: 500Mi
```

The pod definition remains similar:

```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
spec:
  containers:
    - image: alpine
      name: alpine
      command: ["/bin/sh", "-c"]
      args: ["shuf -i 0-100 -n 1 >> /opt/"]
      volumeMounts:
        - mountPath: /opt
          name: data-volume
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: myclaim
```

When you create a PVC with a storage class specified, Kubernetes leverages the defined provisioner to dynamically generate a new persistent disk with the requested size, automatically creating and binding a PV to the PVC.

---

# NETWORKING

## Routing Between Subnets

Now, consider a second network, such as 192.168.2.0, with hosts assigned IPs like 192.168.2.10 and 192.168.2.11. For communication between these two networks, a router is necessary.

A router interconnects two or more networks and holds an IP address in each network—e.g., 192.168.1.1 for the first network and 192.168.2.1 for the second. When a system on network 192.168.1.0 (say, with IP 192.168.1.11) needs to communicate with a system on network 192.168.2.0, it forwards packets to the router.

Each system must be configured with a gateway or specific route entries to ensure that packets reach the intended destination. To view the current routing table, use:

```
route
```

Initially, communication will be limited to the same subnet. To route traffic destined for 192.168.2.0 via the router (with IP 192.168.1.1), add the following route:

```
ip route add 192.168.2.0/24 via 192.168.1.1
```

After adding the route, verifying the routing table should show an entry similar to:

```
route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.2.0     192.168.1.1     255.255.255.0   UG    0      0        0 eth0
```

If a return route is required (for instance, for a host in network 192.168.2.0 to reach a host in 192.168.1.0), add the appropriate route on that system using its corresponding gateway (e.g., 192.168.2.1).

## Configuring Default Routes for Internet Access

To enable internet access (such as reaching external hosts like 172.217.194.0), configure the router as the default gateway. This is done by adding a default route:

```
ip route add default via 192.168.2.1
```

Afterward, your routing table might resemble the following:

```
route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.1.0     192.168.2.1     255.255.255.0   UG    0      0        0 eth0
172.217.194.0   192.168.2.1     255.255.255.0   UG    0      0        0 eth0
default         192.168.2.1     0.0.0.0         UG    0      0        0 eth0
```

### Understanding Default Routes

The "default" or "0.0.0.0" entry indicates that any destination not explicitly listed in the routing table will be directed through the specified gateway.

For scenarios involving multiple routers—such as one handling internet traffic and another managing internal networks—ensure each network has its specific routing entry along with a default route for all other traffic. For example, to route traffic to network 192.168.1.0 via an alternative gateway (192.168.2.2), use:

```
ip route add 192.168.1.0/24 via 192.168.2.2
```

The updated routing table should include:

```
route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         192.168.2.1     0.0.0.0         UG    0      0        0 eth0
192.168.1.0     192.168.2.2     255.255.255.0   UG    0      0        0 eth0
```

If you encounter internet connectivity issues, reviewing the routing table and default gateway configuration is a good troubleshooting practice.

## Configuring a Linux Host as a Router

Consider a scenario with three hosts (A, B, and C) where host B connects to two subnets (192.168.1.x and 192.168.2.x) using two interfaces. For example:
- Host A: 192.168.1.5
- Host B: 192.168.1.6 and 192.168.2.6
- Host C: 192.168.2.5

For host A to communicate with host C, host A must direct traffic aimed at network 192.168.2.0 to host B. On host A, execute:

```
ip route add 192.168.2.0/24 via 192.168.1.6
```

Similarly, host C needs a route for the 192.168.1.0 network via host B (using 192.168.2.6 as the gateway):

```
ip route add 192.168.1.0/24 via 192.168.2.6
```

Once these routes are established, the "network unreachable" error should no longer occur when pinging between host A and host C.

## Enabling IP Forwarding on Linux

Even with the correct routing table, Linux does not forward packets between interfaces by default, as a security measure. This setting is controlled by the IP forwarding parameter in /proc/sys/net/ipv4/ip_forward.

To check the IP forwarding status, run:

```
cat /proc/sys/net/ipv4/ip_forward
```

A return value of 0 indicates that packet forwarding is disabled. To enable forwarding temporarily, run:

```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

Verifying again should now show:

```
cat /proc/sys/net/ipv4/ip_forward
1
```

To ensure this setting persists across reboots, modify /etc/sysctl.conf and add or update the following line:

```
net.ipv4.ip_forward = 1
```

Persisting IP Forwarding: Modifying /etc/sysctl.conf ensures that IP forwarding remains enabled even after a system restart.

---

## DNS

Imagine you have two computers on the same network—Computer A with IP address 192.168.1.10 and Computer B with IP address 192.168.1.11. You can easily ping Computer B from Computer A using its IP address:

```
ping 192.168.1.11
Reply from 192.168.1.11: bytes=32 time=4ms TTL=117
Reply from 192.168.1.11: bytes=32 time=4ms TTL=117
```

Suppose Computer B offers database services. Instead of remembering its IP address, you'll refer to it by a name, for example "db". However, if you immediately try to ping "db" from Computer A, the name remains unrecognized:

```
ping db
ping: unknown host db
```

To make "db" recognizable, add an entry in the /etc/hosts file on Computer A. This informs the system that Computer B (192.168.1.11) is known as "db":

```
cat >> /etc/hosts
192.168.1.11    db
```

After this change, pings to "db" resolve correctly.

You can even create multiple aliases for a single IP address. For instance, you might convince Computer A that Computer B is also known as "www.google.com".

```
ping www.google.com
```

### Centralized DNS Server

To overcome the challenges of managing numerous local host mappings, organizations consolidate all mappings on a centralized DNS server. Suppose your centralized DNS server is at IP address 192.168.1.100. You configure each host to use this server by editing the /etc/resolv.conf file:

```
cat /etc/resolv.conf
nameserver 192.168.1.100
```

Once configured, any hostname that is not found in /etc/hosts is resolved via the DNS server. If an IP address changes, you update the DNS server's records instead of modifying each system individually. Although local /etc/hosts entries—which are useful for test servers—are still honored, they take precedence over DNS queries.

The resolution order is defined in /etc/nsswitch.conf:

```
cat /etc/nsswitch.conf
...
hosts:          files dns
...
```

In this configuration, the system first searches the /etc/hosts file for a hostname. If a match is not found, it then queries the DNS server.

Now, if you try pinging a hostname not found in either /etc/hosts or the DNS server (e.g., www.facebook.com), the resolution fails:

```
cat >> /etc/hosts
192.168.1.115 test


cat /etc/nsswitch.conf
...
hosts:          files dns
...


ping www.facebook.com
ping: www.facebook.com: Temporary failure in name resolution
```

To resolve external domains like Facebook, add a public DNS server (for example, Google's 8.8.8.8) or configure your internal DNS server to forward unresolved queries to a public DNS resolver.

### Domain Names and Structure

Up until now, we have been resolving internal hostnames such as web, db, and nfs. But what is a domain name? A domain name (like www.facebook.com) is composed of parts separated by dots:
- The top-level domain (TLD) appears at the end (e.g., .com, .net, .edu, .org).
- The domain name precedes the TLD (e.g., facebook in www.facebook.com).
- Any segment before the domain name is considered a subdomain (e.g., www).

For instance, consider Google's domain:
- The root is implicit.
- ".com" is the TLD.
- "google" is the main domain.
- "www" is a subdomain.

Subdomains allow organizations to separate services. Examples from Google include maps.google.com for maps, drive.google.com for storage, and mail.google.com for email.
When your organization attempts to access a domain like apps.google.com, the internal DNS server first tries to resolve the name. Failing that, it forwards the request through a hierarchical process: a root DNS server directs it to a .com DNS server, which then points to Google's DNS server. The IP address is returned and cached temporarily to expedite future queries.

### Overview of Common DNS Record Types

DNS records map hostnames to IP addresses and serve various other purposes. Here is an overview of some common DNS record types:

| Record Type | Hostname | Address/Mapping |
|---|---|---|
| A | web-server | Maps hostname to an IPv4 address (e.g., 192.168.1.1) |
| AAAA | web-server | Maps hostname to an IPv6 address (e.g., 2001:0db8:85a3:0000:0000:8a2e:0370:7334) |
| CNAME | food.web-server | Aliases one hostname to another (e.g., aliasing to eat.web-server or hungry.web-server) |

A records handle IPv4 addresses, AAAA records are for IPv6, and CNAME records allow hostname aliasing.

While ping is the most common tool for verifying basic DNS resolution, utilities like nslookup and dig provide more detailed insights.

### Using nslookup and dig

- The nslookup command does not consider /etc/hosts entries and only queries the configured DNS server.
- The dig command offers comprehensive details about DNS queries.

---

## Network Namespaces

Welcome to this detailed guide on network namespaces in Linux. In this guide, we explain how network namespaces provide network isolation—a critical feature in containerized environments like Docker. While the container only sees the processes within its own namespace, the host maintains oversight over all namespaces and can bridge communication between them when required.

On the networking front, the host maintains its own interfaces, ARP tables, and routing configurations—all of which remain hidden from containers. When a container is created, a dedicated network namespace gives it its own virtual interfaces, routing table, and ARP cache.

For example, running the following command on your host:

```
ip link
```

displays the host's interfaces (such as the loopback and Ethernet interfaces). To examine interfaces within a specific network namespace (for example, the "red" namespace), use:

```
ip netns exec red ip link
```

Or with the shorthand using the –n option:

```
ip -n red link
```

Inside the namespace, you typically see only a loopback interface, ensuring that host-specific interfaces (e.g., eth0) remain hidden. This isolation applies similarly to ARP and routing tables.

[To be continued]

---

# HELM

Helm is a package manager and release management tool designed for Kubernetes applications. While Kubernetes excels at orchestrating complex infrastructures, managing a multitude of interdependent YAML files for a single application can quickly become overwhelming. Consider a basic WordPress deployment, which might involve multiple Kubernetes objects such as:
- Deployment: Runs Pods for services like MySQL or web servers.
- Persistent Volume (PV) and Persistent Volume Claim (PVC): Provides storage for databases.
- Service: Exposes the web server to the Internet.
- Secret: Stores sensitive data like admin passwords.
- Job: Handles periodic tasks like backups.

Each of these resources requires its own YAML configuration file and separate kubectl apply commands. Later, when it comes time to upgrade or delete parts of the application, the process involves updating or removing each object—a process that can be both time-consuming and error-prone.

Consider Simplifying Configuration Management

With Helm, you can treat your application as a single package rather than a collection of disparate Kubernetes objects. With Helm, you work with the entire application as a single package. Installing your WordPress app, for example, becomes as simple as executing:

```
$ helm install wordpress
```

## Key Benefits of Helm

**Centralized Configuration:** Customize your application using a single values.yaml file. This file can store essential settings such as:

| Parameter | Description | Example Value |
|---|---|---|
| wordpressUsername | The WordPress admin username | user |
| wordpressPassword | The WordPress admin password (defaults to random) | (optional) |
| wordpressEmail | The admin email address | user@example.com |
| wordpressFirstName | The admin's first name | FirstName |
| wordpressLastName | The admin's last name | LastName |
| wordpressBlogName | The name of the blog | "User's Blog!" |

An example values.yaml file might look like this:

```yaml
wordpressUsername: user
## Application password
## Defaults to a random 10-character alphanumeric string if not set
# wordpressPassword:
## Admin email
wordpressEmail: user@example.com
## First name
wordpressFirstName: FirstName
## Last name
wordpressLastName: LastName
## Blog name
wordpressBlogName: "User's Blog!"
```

**Streamlined Upgrades and Rollbacks:** Helm allows you to:

Upgrade your application effortlessly with:

```
$ helm upgrade wordpress
```

Roll back to a previous release quickly using:

```
$ helm rollback wordpress
```

**Simplified Uninstallation:** Remove all associated resources with a single command:

```
$ helm uninstall wordpress
```

## HELM Components

Helm is primarily composed of a command-line tool installed locally, which you can use to install, upgrade, or roll back releases. Charts—collections of files containing instructions for creating Kubernetes objects—are used by Helm to deploy applications. When you deploy a chart to your Kubernetes cluster, Helm creates a release, representing a specific installation of the application. Each release may have multiple revisions, capturing changes like image upgrades, replica adjustments, or configuration updates.

Helm stores metadata—including information about installed releases, used charts, and revision history—directly into your Kubernetes cluster as secrets. This ensures that metadata is persistent and accessible to all team members, facilitating seamless upgrades and maintenance operations.

## Charts and Templating

Charts are collections of files that contain the instructions for creating Kubernetes objects, making them the backbone of Helm deployments. This article uses two application examples to illustrate various concepts:
- HelloWorld Application: A simple Nginx-based web server with a service for exposure.
- WordPress Site: A more complex application deployment.

The HelloWorld example demonstrates fundamental concepts effectively. In this example, two Kubernetes objects are deployed: a Deployment and a Service. Although similar to standard Kubernetes definitions, you'll notice that parameters like the image name and replica count are templated. These templated values are defined in a separate values.yaml file, which allows you to customize your deployment with minimal changes.

Below is an example of a simple HelloWorld chart:

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: hello-world
```

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: nginx
          image: {{ .Values.image.repository }}
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
```

```yaml
# values.yaml
replicaCount: 1
image:
  repository: nginx
```

Rather than building charts from scratch, you can download pre-built charts from public repositories. Customizing a deployment typically involves modifying the values.yaml file, which serves as the configuration for the Helm chart.

For more complex applications like WordPress, charts can include multiple files and advanced templating features. More detailed explorations of templating and chart structures will be discussed in future lessons. For now, grasping these simple examples will provide you with a solid foundation.

A more advanced templating snippet within a Deployment might look like this:

```yaml
apiVersion: {{ include "common.capabilities.deployment.apiVersion" . }}
kind: Deployment
metadata:
  name: {{ include "common.names.fullname" . }}
  namespace: {{ .Release.Namespace | quote }}
  labels: 
    {{- include "common.labels.standard" . | nindent 4 }}
  {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" (dict "value" .Values.commonLabels "context" $) | nindent 4 }}
  {{- end }}
  {{- if .Values.commonAnnotations }}
  annotations: 
    {{- include "common.tplvalues.render" (dict "value" .Values.commonAnnotations "context" $) | nindent 4 }}
  {{- end }}
spec:
  selector:
    matchLabels: {{- include "common.labels.matchLabels" . | nindent 6 }}
  {{- if .Values.updateStrategy }}
  strategy: {{- toYaml .Values.updateStrategy | nindent 4 }}
  {{- end }}
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
```

## Releases and Multiple Installations

When applying a chart to your cluster, Helm creates a release—a distinct instance of the application. This approach allows you to deploy multiple separate instances of the same chart with unique release names. For example, you can deploy two different WordPress sites using separate releases:

```
# helm install [release-name] [chart]
$ helm install my-site bitnami/wordpress


$ helm install my-SECOND-site bitnami/wordpress
```

Each release is tracked independently, even if they are based on the same chart. This functionality is particularly useful when maintaining different environments, such as having one release for a public-facing website and another for development purposes. Experimentation in the development release can then inform upgrades or changes to the production version.

## Helm Repositories and Artifact Hub

Beyond our basic examples, Helm charts are available for a wide range of applications—from Redis to Prometheus—across numerous public repositories. Providers such as Appscode, Community Operators, TrueCharts, and Bitnami host charts in their repositories, making it easy to deploy various applications.

Instead of visiting multiple repositories separately, you can use the centralized Artifact Hub to search for and manage charts. Artifact Hub currently features over 6,300 packages and highlights charts published by official developers with verified publisher badges for added trustworthiness.

## HELM Charts

Helm Charts act as comprehensive instruction manuals for your deployments. Each chart is a structured collection of files that define an application's configuration and behavior on Kubernetes. For example, the parameters in the values.yaml file enable operators to customize configurations without modifying the underlying templates.

Tip: Use Helm's templating syntax (e.g., `{{ .Values.replicaCount }}`) in your manifests to keep configuration flexible and reusable. All dynamic values are defined in the values.yaml file.

Below is a simple example of Helm template files that create two Kubernetes objects—a Deployment and a Service. The Deployment manages a set of Pods based on a specified image, and the Service exposes these Pods as a NodePort service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: hello-world
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-world
          image: "{{ .Values.image.repository }}"
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
```

To install this chart, run the following command:

```
$ helm install hello-world
```

Notice that values like the image repository and replica count are not hardcoded. Instead, they utilize Helm's templating syntax, which references configurations defined in the values.yaml file. This approach allows you to easily adjust parameters without directly editing the template files.

## Chart Metadata

Every Helm chart includes a Chart.yaml file that contains essential metadata, such as:
- API Version: For Helm 3, set to v2 (Helm 2 charts use v1 or omit this field).
- App Version: Indicates the version of the application deployed.
- Chart Version: Tracks the version of the chart itself, independent of the application version.
- Name and Description: Provide identification and a brief summary of the chart.
- Type: Specifies whether the chart is for an application (default) or is a library chart.
- Dependencies: Declare any external charts that the chart relies on.
- Additional Fields: Optional fields like keywords, maintainers, home, and icon help with discovery and branding.

Below is an example that combines Kubernetes manifest templates with chart metadata:

```yaml
# Service and Deployment templates
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: hello-world
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-world
          image: "{{ .Values.image.repository }}"
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
---
# values.yaml snippet
replicaCount: 1
image:
  repository: nginx
---
# Chart.yaml snippet
apiVersion: v2
appVersion: "1.16.0"
name: hello-world
description: A web application
type: application
```

Again, the chart can be installed with:

```
$ helm install hello-world
```

### Example: WordPress Chart

For a more complex use case, consider a WordPress chart that depends on additional services like MariaDB. Below is an example of a Chart.yaml file for a WordPress deployment:

```yaml
apiVersion: v2
appVersion: 5.8.1
version: 12.1.27
name: wordpress
description: Web publishing platform for building blogs and websites.
type: application
dependencies:
  - condition: mariadb.enabled
    name: mariadb
    repository: https://charts.bitnami.com/bitnami
    version: 9.x.x
keywords:
  - application
  - blog
  - wordpress
maintainers:
  - email: containers@bitnami.com
    name: Bitnami
home: https://github.com/bitnami/charts/tree/master/bitnami/wordpress
icon: https://bitnami.com/assets/stacks/wordpress/img/wordpress-stack-220x234.png
```

## Helm Chart Directory Structure

A typical Helm chart directory includes the following components:
- `templates/`: Contains all the manifest templates (e.g., Deployment, Service).
- `values.yaml`: Defines default configuration values.
- `Chart.yaml`: Holds metadata about the chart.
- `charts/`: Optionally includes dependent charts (e.g., the MariaDB chart for WordPress).
- Other optional files such as LICENSE or README for additional information.

## Deploying a Chart from a Repository

To deploy the WordPress chart from the Bitnami repository, execute the following commands:

```
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm install my-release bitnami/wordpress
```

You can verify your installation using similar commands:

```
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm install my-release bitnami/wordpress
```

## Helm Basic Commands

```
helm --help
```

### Managing Chart Repositories

Exploring repository-related commands is straightforward. Running:

```
helm repo --help
```

## Deploying an Application on Kubernetes

Imagine you want to launch a WordPress website on Kubernetes using a chart from an online repository, such as Artifact Hub. Charts with an official or verified publisher badge ensure authenticity and reliability. Follow these steps:

Note: First, add the Bitnami repository:

```
$ helm repo add bitnami https://charts.bitnami.com/bitnami
```

Note: Then, install the WordPress chart:

```
$ helm install my-release bitnami/wordpress
```

## Managing Helm Releases

Once a chart is deployed, it is managed as a release. To view all existing releases, run:

```
$ helm list
```

```
$ helm uninstall my-release
```

This command cleans up all Kubernetes objects that were created during deployment.

## Managing Local Helm Repository Data

To view your configured chart repositories, execute:

```
$ helm repo list
```

Over time, locally cached repository data can become outdated. To refresh this information, run:

```
$ helm repo update
```

---

## Customizing Chart Parameters

In this guide, you'll learn how to customize chart parameters during a Helm chart installation.

### Overriding Default Values with Command-Line Parameters

```
helm install --set wordpressBlogName="Helm Tut" my-release bitnami/wordpress
```

### Using a Custom Values File

For numerous parameter overrides, maintaining a custom values file is more efficient than using multiple --set options. Follow these steps:

Create a file named custom-values.yaml with your custom configurations:

```yaml
# custom-values.yaml snippet
wordpressBlogName: Helm Tut
wordpressEmail: john@example.com
```

Deploy the chart using the custom values file by running:

```
$ helm install my-release bitnami/wordpress --values custom-values.yaml
```

This instructs Helm to apply the configuration from custom-values.yaml, thereby overriding the corresponding default values.

### Modifying the Built-in values.yaml File

If you wish to modify the built-in values.yaml within the chart itself, you can do so by following these steps:

Use helm pull to download the chart in an archived (compressed) form:

```
$ helm pull bitnami/wordpress
```

To automatically uncompress the chart into a directory, use the --untar option:

```
$ helm pull --untar bitnami/wordpress
```

List the directory to view the chart files, including values.yaml:

```
$ ls
wordpress


$ ls wordpress
ci                templates          Chart.lock
Chart.yaml        README.md          values.schema.json
values.yaml
```

Open and edit the values.yaml file in your preferred text editor to set your custom configurations.

Install the chart locally by referencing the modified chart path:

```
$ helm install my-release ./wordpress
```

In this command, ./ indicates the current directory, and Helm installs the chart using your modified files.

## Creating and Managing Releases

When you install a Helm chart, a release is created. Each release is like an application package—a collection of related Kubernetes objects. Since Helm tracks all objects associated with a release, it allows you to upgrade, downgrade, or uninstall a release without affecting other deployments. For instance, even if you deploy the same chart twice, each release remains independent:

```
$ helm install my-site bitnami/wordpress
$ helm install my-SECOND-site bitnami/wordpress
```

e.g. installing an older version of nginx:

```
$ helm install nginx-release bitnami/nginx --version 7.1.0
```

upgrading :

```
helm upgrade nginx-release bitnami/nginx
```

After upgrading, verify the updates with the following commands:

```
$ helm list
```

Check the revision history to get more insights into the changes:

```
$ helm history nginx-release
```

rollback :

```
helm rollback nginx-release 1
```
