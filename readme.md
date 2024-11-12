**Running Multiple Kubernetes Clusters Locally Using Minikube and vClusters: A Guide to Multi-Environment Simulation**
======================================================================================================================

**Introduction**

Imagine simulating multiple Kubernetes environments on your local machine — each with its own workloads, networking, and ingress configurations — all while using a single Minikube instance. In this guide, we’ll explore how to achieve this by creating virtual clusters (vClusters) within Minikube.

**Why Use vClusters?**

Using vClusters within Minikube is a resource-efficient way to simulate multiple Kubernetes clusters on a single machine. This method is perfect for local testing, development, and CI pipelines without the overhead of running multiple full-scale Kubernetes clusters.

**Project Purpose**

The goal of this project is to simulate a multi-cluster Kubernetes environment locally by creating a single Minikube cluster, within which we’ll deploy two virtual clusters (vClusters) using the vCluster tool. Each vCluster will contain its own isolated Nginx deployment and service, allowing us to manage separate workloads as if they were in distinct Kubernetes clusters. To facilitate traffic routing, we’ll configure Ingress resources in the main Minikube cluster to direct traffic based on URL paths to each vCluster. This setup provides a flexible environment for testing and experimenting with multi-cluster setups, ideal for DevOps workflows and local development.

**Prerequisites**

Before we begin, ensure you have the following tools installed:

*   [kubectl](https://kubernetes.io/docs/reference/kubectl/)
*   [minikube](https://minikube.sigs.k8s.io/docs/start/)
*   [vcluster](https://www.vcluster.com/)
*   [docker](https://www.docker.com/)

**Getting Started**

**Step 1: Start Minikube**

Start Minikube by running:

```
minikube start
```

**Note**: You can specify memory and CPU allocation with the \` — memory\` and \` — cpus\` flags, for example:

```
minikube start — cpus=4 — memory=8192
```

**Step 2: Enable the Ingress Addon**

Enable the Ingress addon in Minikube, which will allow you to define and manage Ingress resources.

```
minikube addons enable ingress
```

**Step 3: Run the Minikube Tunnel**

In a new terminal, run:

```
minikube tunnel
```

The \`minikube tunnel\` command sets up routing and load balancing for Ingress resources by exposing Ingress services to your local network. This allows you to access services in Minikube as if they were in a cloud environment.

**Step 4: Create a Namespace for the First vCluster**

Next, create a namespace to isolate your first virtual cluster within Minikube:

```
kubectl create namespace vcluster-1  
kubectl create namespace vcluster-2
```

**Step 5: Set Up the First vCluster**

Create and install the control plane and required components for `vcluster1`and `vcluster2`using the following command:

```
vcluster create vcluster-1 -n vcluster-1  
vcluster create vcluster-2 -n vcluster-2
```

This command launches a new virtual cluster within the \`vcluster-1\` namespace, setting up a separate Kubernetes control plane (API server, scheduler, etc.) that operates independently from Minikube’s main cluster. Each vCluster has its own control plane while sharing the underlying Minikube worker node resources.

**Step 6: Connect to the New vCluster**

To interact with vcluster-1 and vcluster-2, connect to it with the command:

```
vcluster connect vcluster-1 -n vcluster-1  
vcluster connect vcluster-2 -n vcluster-2
```

Next, we’ll create a deployment and service for Nginx in each vCluster. To help verify that each request routes to the correct cluster and namespace, we’ll customize the Nginx deployment to display a message indicating the namespace (insdie the cluster) it’s coming from.

Steps for Setting Up Nginx in vCluster-1
----------------------------------------

1.  **Connect to vCluster-1**  
    You can connect using one of the following commands:

```
kubectl config get-contexts  
kubectl config use-context <Name of the vCluster-1 Context>  
or  
vcluster connect vcluster-1 -n vcluster-1
```

2\. Create a New Namespace

```
kubectl create ns vcluster-1
```

3\. Deploy Nginx with a Customized Message

This configuration deploys an Nginx pod that displays “hello from namespace vcluster1” in the default Nginx webpage.

> Deployment YAML (vCluster-1)

```
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: nginx-deployment  
  namespace: vcluster-1  
spec:  
  replicas: 1  
  selector:  
    matchLabels:  
      app: nginx  
  template:  
    metadata:  
      labels:  
        app: nginx  
    spec:  
      containers:  
      - name: nginx  
        image: nginx:latest  
        env:  
        - name: NAMESPACE  
          valueFrom:  
            fieldRef:  
              fieldPath: metadata.namespace  
        command: \["/bin/sh", "-c"\]  
        args:  
          - echo "hello from namespace $NAMESPACE" > /usr/share/nginx/html/index.html &&  
            nginx -g 'daemon off;'  
        ports:  
        - containerPort: 80
```

> Service YAML (vCluster-1)

```
apiVersion: v1  
kind: Service  
metadata:  
  name: nginx-service  
  namespace: vcluster-1  
spec:  
  selector:  
    app: nginx  
  ports:  
    - protocol: TCP  
      port: 80  
      targetPort: 80  
  type: ClusterIP
```

4\. Verify the Nginx Pod is Running

```
kubectl get pods -n vcluster-1
```

Steps for Setting Up Nginx in vCluster-2
----------------------------------------

1.  Connect to vCluster-2

```
kubectl config get-contexts  
kubectl config use-context <Name of the vCluster-2 Context>  
or  
vcluster connect vcluster-2 -n vcluster-2  

```

2\. Create a New Namespace

```
kubectl create ns vcluster-2
```

3\. Deploy Nginx with a Customized Message for vCluster-2

> Deployment YAML (vCluster-2)

```
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: nginx-deployment  
  namespace: vcluster-2  
spec:  
  replicas: 1  
  selector:  
    matchLabels:  
      app: nginx  
  template:  
    metadata:  
      labels:  
        app: nginx  
    spec:  
      containers:  
      - name: nginx  
        image: nginx:latest  
        env:  
        - name: NAMESPACE  
          valueFrom:  
            fieldRef:  
              fieldPath: metadata.namespace  
        command: \["/bin/sh", "-c"\]  
        args:  
          - echo "hello from namespace $NAMESPACE" > /usr/share/nginx/html/index.html &&  
            nginx -g 'daemon off;'  
        ports:  
        - containerPort: 80
```

> Service YAML (vCluster-2)

```
apiVersion: v1  
kind: Service  
metadata:  
  name: nginx-service  
  namespace: vcluster-2  
spec:  
  selector:  
    app: nginx  
  ports:  
    - protocol: TCP  
      port: 80  
      targetPort: 80  
  type: ClusterIP
```

4\. Verify the Nginx Pod is Running

```
kubectl get pods -n vcluster-2
```

Couple Wrods about the Services..
=================================

When we create a service within a vCluster, Minikube automatically creates a corresponding service in its namespace that points to the internal service inside the vCluster. This enables our main cluster (Minikube) to interact with services in each vCluster as if they were native Kubernetes services.

We can now use these service names to define routing rules in an Ingress.

> While it’s possible to set up a single Ingress to route traffic across multiple vClusters (located in different namespaces), not all Ingress controllers support cross-namespace routing by default. The NGINX Ingress Controller and some third-party controllers may support it through specific annotations, but compatibility varies.
> 
> **Security Considerations:** Enabling cross-namespace routing can expose services to broader access patterns, so it’s essential to review network policies and access controls carefully to maintain proper isolation between environments.

Now, let’s switch our Kubernetes context back to the Minikube cluster

```
kubectl config use-context minikube
```

Viewing Services Across All Namespaces

```
kubectl get svc -A
```

This will show two key services related to our vClusters

```
vcluster-1      nginx-service-x-vcluster1-x-vcluster-1   ClusterIP   10.107.210.172  
vcluster-2      nginx-service-x-vcluster2-x-vcluster-2   ClusterIP   10.100.245.244 
```

These services are internal endpoints for accessing the Nginx deployments in each virtual cluster. Since each vCluster operates as a fully isolated Kubernetes environment, they can run similar workloads, such as Nginx services, without port or naming conflicts — even when running on the same Minikube instance. These services are crucial for isolating network traffic, as they route requests specifically to each vCluster’s Nginx deployment.

> To verify connectivity, you can `curl` these services from within any pod in the Minikube cluster (e.g., using a simple pod like `shpod`). You should see the response from the Nginx service within each vCluster, confirming the internal routing is correctly set up.

run shpod pod and curl the service

```
kubectl run shpod --rm -i --tty --image=alpine --namespace=default -- /bin/sh
```

we can see that the service running in our minikube cluster point to the correct pod in vcluster.

**Creating an Ingress for Each vCluster (Namespace)**
=====================================================

In this step, we will create an Ingress for each virtual cluster (vCluster). Each Ingress will route traffic to the respective Nginx service based on the defined path.

`ingress.yaml` for **vcluster-1**:

```
apiVersion: networking.k8s.io/v1  
kind: Ingress  
metadata:  
  name: nginx-ingress  
  namespace: vcluster-1  
  annotations:  
    nginx.ingress.kubernetes.io/rewrite-target: /  
spec:  
  rules:  
  - host: vcluster.minikube.local  
    http:  
      paths:  
      - path: /v1  
        pathType: Prefix  
        backend:  
          service:  
            name: nginx-service-x-vcluster-1-x-vcluster-1  
            port:  
              number: 80
```

`ingress.yaml` for **vcluster-2**

```
apiVersion: networking.k8s.io/v1  
kind: Ingress  
metadata:  
  name: nginx-ingress  
  namespace: vcluster-2  
  annotations:  
    nginx.ingress.kubernetes.io/rewrite-target: /  
spec:  
  rules:  
  - host: vcluster.minikube.local  
    http:  
      paths:  
      - path: /v2  
        pathType: Prefix  
        backend:  
          service:  
            name: nginx-service-x-vcluster-2-x-vcluster-2  
            port:  
              number: 80
```

> Before proceeding, ensure that the Minikube tunnel is active

Update `/etc/hosts` or Local DNS

To make sure that `vcluster.minikube.local` resolves correctly to the Minikube cluster's IP address, add the Minikube IP to your `hosts` file.

If you’re using Minikube in **Docker mode**, add `127.0.0.1` as the IP for `vcluster.minikube.local`.

Testing with Curl  

For vCluster-1

```
curl http://vcluster.minikube.local/v1
```

For vCluster-2

```
curl http://vcluster.minikube.local/v2
```

This setup allows for simulating multiple isolated clusters within a single Minikube instance, providing an environment where developers can test cross-cluster routing and ingress configurations locally.

medium → [medium](https://medium.com/@itaybeyder_76401/running-multiple-kubernetes-clusters-locally-using-minikube-and-vclusters-a-guide-to-3482f1ed61f8)
