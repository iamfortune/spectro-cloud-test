# Deploy Applications with Kind

[Kubernetes](https://kubernetes.io/) is an open-source platform that automates the deployment, scaling, and management of containerized applications. It has become the standard for running applications in production environments. However, setting up a full Kubernetes cluster can be resource-intensive and complex, especially when you're learning or testing.

[Kind](https://kind.sigs.k8s.io/) (Kubernetes in Docker) solves this problem by running Kubernetes clusters inside Docker containers, making it perfect for local development and learning.

In this tutorial, you will deploy a simple web application to a local Kubernetes cluster using Kind. You will learn how to create a cluster, deploy an application using Kubernetes manifests, expose the application through a service, and access it from your local machine.
By the end of this tutorial, you will understand the basic workflow of deploying applications to Kubernetes.

# Prerequisites

Before you begin, ensure you have the following installed:

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) or Docker Engine running on your local machine
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) version 0.17.0 or later
- [kubectl](https://kubernetes.io/docs/tasks/tools/) command-line tool

You should also have basic familiarity with using a terminal or command line interface.

# Create a Kubernetes Cluster

Kind allows you to create lightweight Kubernetes clusters that run entirely in Docker containers. This makes it easy to spin up and tear down clusters without consuming significant system resources.

## Start the Cluster

Open your terminal and issue the following command:

```shell
kind create cluster
```

The cluster creation process takes approximately 1-2 minutes. You will see output similar to the following:

```shell
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.34.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a nice day! 👋
```

Once the cluster is created, Kind automatically configures kubectl to connect to your new cluster. The cluster is named "kind" by default.

**What just happened?** Kind created a [Docker container](https://www.docker.com/resources/what-container/) that acts as a Kubernetes node. Inside this container runs all the components needed for a functional Kubernetes cluster, including the control plane that manages your applications.

## Verify Cluster Connectivity

Before deploying an application, verify that kubectl can communicate with your Kubernetes cluster. Issue the following command:

```shell
kubectl cluster-info --context kind-kind
```

You should see output displaying the control plane address and CoreDNS service:

```shell
Kubernetes control plane is running at https://127.0.0.1:xxxxx
CoreDNS is running at https://127.0.0.1:xxxxx/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

This confirms that your cluster is running and kubectl is properly configured to communicate with it.

**Common Issue**: If you see a connection error, ensure Docker is running and the cluster creation completed successfully. You can list your Kind clusters with `kind get clusters`.

# Deploy an Application

Now that your cluster is ready, you will deploy a simple web application. Kubernetes uses YAML manifest files to define the desired state of your applications. These manifests tell Kubernetes what you want to run and how you want it configured.

## Understand Kubernetes Resources

Before creating your manifest, understand two key Kubernetes concepts:

**[Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)**: A Deployment manages a set of identical [pods](https://kubernetes.io/docs/concepts/workloads/pods/) (the smallest deployable units in Kubernetes). It ensures the desired number of pod replicas are always running. If a pod crashes, the Deployment automatically creates a new one to replace it.

**[Service](https://kubernetes.io/docs/concepts/services-networking/service/)**: A Service provides a stable network endpoint to access your pods. Pods are ephemeral and can be replaced at any time, which changes their IP addresses. Services solve this problem by providing a consistent way to reach your application.

## Create the Application Manifest

Create a file named **app.yaml** in your working directory and add the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - image: gcr.io/google-samples/hello-app:1.0
        name: hello-app
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: web
  name: web
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: web
  type: NodePort
```

The manifest contains two Kubernetes resources separated by `---`:

**Deployment section**:
- `replicas: 1` - Runs one copy of your application
- `containers` - Specifies the container image (`hello-app`) to run
- `labels` - Used to identify and group related resources

**Service section**:
- `type: NodePort` - Exposes the application by opening a port on the cluster node. This makes the application accessible from outside the cluster, which is necessary for local development with Kind.
- `targetPort: 8080` - The port your container listens on
- `port: 8080` - The port the Service listens on
- `selector` - Identifies which pods receive traffic (those with label `app: web`)

**Why NodePort?** Kubernetes offers several [Service types](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types). NodePort is suitable for local development because it exposes your application through the node's IP address. In production environments, you would typically use LoadBalancer or ClusterIP with an Ingress controller.

## Apply the Manifest

Deploy the application to your cluster by applying the manifest:

```shell
kubectl apply -f app.yaml
```

You will see confirmation that both resources were created:

```shell
deployment.apps/web created
service/web created
```

Kubernetes now begins pulling the container image and creating the pod. This may take a few moments depending on your internet connection.

## Verify the Deployment

Check that your pod is running successfully:

```shell
kubectl get pods
```

You should see output similar to:

```shell
NAME                   READY   STATUS    RESTARTS   AGE
web-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
```

**Understanding the output**:
- `READY 1/1` - One container is ready out of one total container
- `STATUS Running` - The pod is successfully running
- `RESTARTS 0` - The pod has not crashed and restarted

**Troubleshooting**: If the STATUS shows `ImagePullBackOff` or `ErrImagePull`, Kubernetes cannot download the container image. Check your internet connection. If STATUS shows `CrashLoopBackOff`, the container is failing to start—check the logs with `kubectl logs <pod-name>`.

# Access the Application

Your application is now running inside the cluster, but you need a way to access it from your local machine. Kubernetes provides port forwarding functionality to create a network tunnel from your local environment to a pod running in the cluster.

## Understand Port Forwarding

[Port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) creates a direct connection from your computer to a specific pod. This is useful for:
- Testing applications during development
- Debugging issues with specific pods
- Accessing services not exposed externally

For production environments, you would typically use a Service with type LoadBalancer or an Ingress controller instead of port forwarding.

## Forward the Application Port

First, retrieve the pod name and store it in a variable for easy reference:

```shell
PODNAME=$(kubectl get pods --template '{{range .items}}{{.metadata.name}}{{end}}' --selector=app=web)
```

This command uses a Go template to extract the pod name and assigns it to the `PODNAME` variable. The selector filters pods by their label.

Verify the variable contains the pod name:

```shell
echo $PODNAME
```

Next, start port forwarding to expose the application on your local machine:

```shell
kubectl port-forward $PODNAME 8080:8080
```

You will see output indicating the forwarding is active:

```shell
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

The format is `local-port:pod-port`. Traffic sent to `localhost:8080` on your machine now forwards to port 8080 inside the pod.

The port forwarding remains active in your terminal session. Keep this terminal window open while testing the application.

## Test the Application

Open a web browser and visit http://localhost:8080. You should see a page displaying:

```
Hello, world!
Version: 1.0.0
Hostname: web-xxxxxxxxxx-xxxxx
```

The hostname matches your pod name, confirming you are connected to the correct container.

Congratulations! You have successfully deployed and accessed your first application on Kubernetes.

To stop port forwarding, return to your terminal and press `Ctrl+C`.

# Cleanup

When you finish experimenting with your cluster, you should remove all resources to free up system resources. Kind makes cleanup simple by removing the entire cluster at once.

Delete the Kind cluster with the following command:

```shell
kind delete cluster
```

You will see confirmation that the cluster was deleted:

```shell
Deleting cluster "kind" ...
Deleted nodes: ["kind-control-plane"]
```

This command removes the Docker container running your cluster and all resources within it, including your deployed application, Service, and any other objects you created.

**Note**: If you created multiple Kind clusters, you can specify which one to delete with `kind delete cluster --name <cluster-name>`.

# Next Steps

In this tutorial, you created a local Kubernetes cluster using Kind, deployed a containerized application using a Deployment and Service, and accessed the application through port forwarding. You learned the fundamental workflow of deploying applications to Kubernetes: defining resources with manifests, verifying deployments, and exposing services.

To continue your Kubernetes journey, consider exploring these topics:

- Learn about [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) and how to scale applications by changing replica counts
- Explore different [Service types](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types) such as ClusterIP and LoadBalancer
- Practice writing your own application manifests for multi-container deployments
- Discover how [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) and [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) help manage application configuration
- Experiment with [kubectl commands](https://kubernetes.io/docs/reference/kubectl/) to inspect and manage your resources

For production-grade Kubernetes management across multiple environments, explore [Spectro Cloud's Palette platform](https://www.spectrocloud.com/), which simplifies deploying and managing Kubernetes clusters at scale.