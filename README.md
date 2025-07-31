# Ways to Run and Access an NGINX Container on Kubernetes

### Introduction
Kubernetes offers multiple types of workloads to manage containerized applications. In this post, we’ll explore how to run an NGINX container using different Kubernetes objects like Pods, ReplicaSets, Deployments, StatefulSets, DaemonSets, Helm, and more — all with steps to expose them for access.

## Prerequisites

- A running Kubernetes cluster (Minikube, KIND, EKS, GKE, or AKS) in our case we will use minikube cluster created locally
- `kubectl` CLI configured
- `Helm` installed (for Helm-based setup)

### 1) Pod
A Pod is the most basic unit of deployment in Kubernetes. It represents a single instance of a running process, and in this case, it can be used to run a single NGINX container. Pods are ideal for quick testing or experimentation but aren’t suitable for production workloads since they don’t support replication, self-healing, or rolling updates. You can deploy a Pod using a simple YAML manifest and access it via port-forwarding or by creating a NodePort service temporarily. 

A) **Imperative way** (via kubectl commands)
You tell Kubernetes exactly what to do right now.
```bash
kubectl run nginx-pod --image=nginx --port=80 -n default
```
Check if the pod is running:
```bash
$ kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          23s
```
Expose it:
```bash
kubectl expose pod nginx-pod --type=NodePort --port=80 --name=nginx-service
```
This will create a service of type NodePort and expose
```bash
$ kubectl get svc
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP   10.96.0.1      <none>        443/TCP        9m22s
nginx-service   NodePort    10.96.83.111   <none>        80:32282/TCP   6s
```

Now, let’s try to access the Nginx application through the browser.
```bash
$ minikube service nginx-service --url
http://127.0.0.1:51986
! Because you are using a Docker driver on windows, the terminal needs to be open to run it.
```

Accessing the Nginx application is possible by visiting the provided URL, which is http://127.0.0.1:51986.
![alt text](/images/nginx.png)

Cleanup:
```bash
# Delete the service
kubectl delete service nginx-service
# Delete the pod
kubectl delete pod nginx-pod
```

b) **Declarative way** (via YAML manifests)
You describe the desired state in a YAML file and apply it.

`nginx-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

Apply it:
```bash
kubectl apply -f nginx-pod.yaml
```

Check if the pod is running:
```bash
$ kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          6s
```

Create a YAML file for nginx service:
`nginx-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```
apply it:
```bash
 kubectl apply -f nginx-service.yaml
```
This will create a service of type NodePort and expose
```bash
$ kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        17m
nginx-svc    NodePort    10.108.129.175   <none>        80:30288/TCP   8s
```

And you can access the pod in same way as earlier

Cleanup:
```bash
# Delete the service using the YAML file
kubectl delete -f nginx-service.yaml
# Delete the pod using the YAML file
kubectl delete -f nginx-pod.yaml
```

### 2) ReplicationController
ReplicationController (RC) is an older Kubernetes controller used to maintain a fixed number of identical Pods running at any given time. Though largely replaced by ReplicaSets and Deployments, RCs still exist and can be used to demonstrate how Kubernetes ensures availability by replacing failed Pods. To deploy NGINX using an RC, you define the number of replicas and a Pod template. Once deployed, you can expose the RC using a NodePort service to access NGINX from outside the cluster. It's mainly used today for legacy compatibility.

Create a YAML file:
`nginx-replication-controller.yaml`
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc
spec:
  replicas: 2
  selector:
    app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```
apply it:
```bash
kubectl apply -f nginx-replication-controller.yaml -n default
```
check if its running required number of pods:

```bash
$ kubectl get rc
NAME       DESIRED   CURRENT   READY   AGE
nginx-rc   2         2         2       32s

$ kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
nginx-rc-j7fw2   1/1     Running   0          39s
nginx-rc-x9gdk   1/1     Running   0          39s
```

Expose the it using service (we can use earlier nginx-service.yaml). The service selects the pods running with the label selector. The pods can be accessed using similar way mentioned earlier.

Cleanup:
```bash
kubectl delete -f nginx-replication-controller.yaml
```

### 3) ReplicaSet

ReplicaSet is the next evolution of ReplicationController, providing more flexible label selectors and tighter integration with modern workloads. While you can use a ReplicaSet directly to manage NGINX replicas, it’s commonly used under the hood by Deployments. This method ensures the desired number of NGINX Pods are always running. You can expose the ReplicaSet using a NodePort service to access it externally. Though you can use ReplicaSets directly, it's best practice to use them via Deployments for better lifecycle management and upgrade strategies.

Create a YAML filefor replica set:
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
```
and nginx-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```
Apply them:
```bash
kubectl apply -f nginx-replicaset.yaml -n default
kubectl apply -f nginx-service.yaml -n default
```
Check if its running:
```bash
$ kubectl get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-replicaset-9w775   1/1     Running   0          35s
pod/nginx-replicaset-q7kw4   1/1     Running   0          35s

NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/kubernetes      ClusterIP   10.96.0.1      <none>        443/TCP        50m
service/nginx-service   NodePort    10.103.27.19   <none>        80:31199/TCP   19s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-replicaset   2         2         2       35s
```

You can access the pod as earlier.

Cleanup:
```bash
kubectl delete -f nginx-replicaset.yaml -n default
kubectl delete -f nginx-service.yaml -n default
```

### 4) Deployment
Deployment is the most widely used workload controller in Kubernetes. It manages ReplicaSets and provides declarative updates for Pods and ReplicaSets, making it perfect for production scenarios. Deploying NGINX via a Deployment allows you to scale easily, perform rolling updates, and roll back if something goes wrong. This setup is ideal for running web servers in a highly available manner. After deploying, you can expose it using a Service of type NodePort, LoadBalancer, or via an Ingress to access NGINX from outside the cluster.

Create nginx-deployment.yaml:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
```
and nginx-service.yaml :
```bash
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```
Apply them:
```bash
kubectl apply -f nginx-deployment.yaml -n default
kubectl apply -f nginx-service.yaml -n default
```

Check if everythins is running:
```bash
$ kubectl get all
NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-58cdc7b878-bf7gh   1/1     Running   0          81s
pod/nginx-deployment-58cdc7b878-w82sr   1/1     Running   0          81s

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        55m
service/nginx-service   NodePort    10.99.103.182   <none>        80:30357/TCP   65s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   2/2     2            2           81s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-58cdc7b878   2         2         2       81s
```

Nginx can be accessed as mentioned earlier.

Cleanup:
```bash
kubectl delete -f nginx-deployment.yaml -n default
kubectl delete -f nginx-service.yaml -n default
```

### 5) StatefulSet
StatefulSet is designed for managing stateful applications that require persistent storage and stable network identities. While NGINX is typically stateless, deploying it via a StatefulSet helps demonstrate scenarios where you need unique pod hostnames, such as with caches or databases. Each pod gets a sticky identity (e.g., nginx-0, nginx-1) and can be accessed via stable DNS addresses within the cluster. For external access, you need to configure an Ingress or LoadBalancer. Though not typical for NGINX, it's a good learning exercise for understanding StatefulSet behavior.

Create nginx-statefulset.yaml
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-sts
  labels:
    app: nginx
spec:
  replicas: 2
  serviceName: nginx
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
```
Create nginx-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

This StatefulSet maintains two replicas and provides each Pod with a stable, unique network identity — a key requirement for stateful applications.

The serviceName field in a StatefulSet refers to a headless service that facilitates DNS-based discovery and direct communication with each individual Pod. It defines the service responsible for managing the network identity of the Pods within the StatefulSet.

Apply them:
```bash
kubectl apply -f nginx-statefulset.yaml -n default
kubectl apply -f nginx-service.yaml -n default
```

Check if everything is running:
```bash
$ kubectl get all
NAME              READY   STATUS    RESTARTS   AGE
pod/nginx-sts-0   1/1     Running   0          13s
pod/nginx-sts-1   1/1     Running   0          9s

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        75m
service/nginx-service   NodePort    10.96.136.227   <none>        80:30171/TCP   6s

NAME                         READY   AGE
statefulset.apps/nginx-sts   2/2     13s
```

Nginx can be accessed as mentioned earlier.

CLeanup:
```bash
kubectl delete -f nginx-statefulset.yaml -n default
kubectl delete -f nginx-service.yaml -n default
```

### 6) DaemonSet
A DaemonSet ensures that a copy of a Pod (in this case, running NGINX) runs on every node in your cluster. This is useful when you want a service like NGINX to be available on all nodes, perhaps for node-local routing, logging, or monitoring purposes. You can configure each NGINX container with a hostPort to make it accessible via the host's network interface. Alternatively, use a NodePort service to reach any of the NGINX instances. This method is not typical for web servers in production, but it’s highly useful for infrastructure-level components.

Create nginx-daemonset.yaml
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemonset
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
```
Create nginx-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```
apply them:
```bash
kubectl apply -f nginx-daemonset.yaml -n default
kubectl apply -f nginx-service.yaml -n default
```

Check if everything is running:
```bash
$ kubectl get all
NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-daemonset-nbf2h   1/1     Running   0          11s

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        83m
service/nginx-service   NodePort    10.111.183.20   <none>        80:32707/TCP   4s

NAME                             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/nginx-daemonset   1         1         1       1            1           <none>          11s
```
and you can access nginx as mentioned above.

Cleanup:
```bash
kubectl delete -f nginx-daemonset.yaml -n default
kubectl delete -f nginx-service.yaml -n default
```
### 7) Helm
Helm is the package manager for Kubernetes that simplifies deployment through reusable, templatized YAML charts. Instead of manually creating manifests for each component, you can deploy NGINX using a Helm chart from a public repository like Bitnami. This method is ideal for teams managing complex applications or infrastructure, as Helm supports values overrides, upgrades, and rollbacks. Deploying NGINX via Helm is fast and efficient, and it also makes version control and customization easier. Services created by the chart often include a LoadBalancer or Ingress to expose NGINX automatically.

A) Official HELM chart

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install my-nginx bitnami/nginx
```

```bash
$ helm install my-nginx bitnami/nginx
NAME: my-nginx
LAST DEPLOYED: Thu Jul 31 12:49:13 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: nginx
CHART VERSION: 21.0.8
APP VERSION: 1.29.0
```

Check if everything is running:
```bash
$ kubectl get all
NAME                            READY   STATUS    RESTARTS   AGE
pod/my-nginx-864d6c4cfb-xmsl9   1/1     Running   0          99s

NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP                      107m
service/my-nginx     LoadBalancer   10.109.216.60   <pending>     80:32573/TCP,443:31138/TCP   99s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-nginx   1/1     1            1           99s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/my-nginx-864d6c4cfb   1         1         1       99s
```

Minikube doesn’t support cloud LoadBalancer services, so you’ll need to use Minikube’s tunnel or service exposure.
```bash
$ minikube service my-nginx --url
http://127.0.0.1:58205
http://127.0.0.1:58206
! Because you are using a Docker driver on windows, the terminal needs to be open to run it.
```

Cleanup:
``bash
$ helm uninstall my-nginx
``

B) Create your own HELM chart

Create helm chart using:
```bash
helm create nginx-chart
```

This will create a new directory in your project called nginx-chart with the follwing structure:
```bash
\NGINX-CHART
│   .helmignore
│   Chart.yaml
│   values.yaml
├───charts
└───templates
    │   deployment.yaml
    │   hpa.yaml
    │   ingress.yaml
    │   NOTES.txt
    │   service.yaml
    │   serviceaccount.yaml
    │   _helpers.tpl
    └───tests
            test-connection.yaml
```

Here remove all files under templates/ and update with the below files.

```bas
\NGINX-CHART
│   .helmignore
│   Chart.yaml
│   values.yaml
│
├───charts
└───templates
        deployment.yaml
        NOTES.txt
        service.yaml
        _helpers.tpl
```

deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    {{- include "nginx-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "nginx-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "nginx-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: nginx
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default "latest" }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
```

service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-service
  labels:
    {{- include "nginx-chart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 80
      protocol: TCP
      name: http
  selector:
    {{- include "nginx-chart.selectorLabels" . | nindent 4 }}
```

_helpers.tpl
```bash
{{/*
Expand the name of the chart.
*/}}
{{- define "nginx-chart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "nginx-chart.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "nginx-chart.labels" -}}
helm.sh/chart: {{ include "nginx-chart.chart" . }}
{{ include "nginx-chart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "nginx-chart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "nginx-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

NOTES.txt
```bash
1. Get the application URL by running this commands:
  minikube service {{ .Release.Name }}-service --url
```

values.yaml
```yaml
# Default values for nginx-chart.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

# This will set the replicaset count
replicaCount: 2

# This sets the container image
image:
  repository: nginx
  # This sets the pull policy for images.
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: ""

 # This is for setting up a service
service:
  # This sets the service type
  type: NodePort
  # This sets the ports
  port: 80
```

Chart.yaml
```yaml
# nginx-chart/Chart.yaml
apiVersion: v2
name: nginx-chart
description: A simple Helm chart for NGINX
version: 0.1.0
appVersion: "latest"
```

 Install the Chart
```bash
$ helm install nginx nginx-chart -n default
NAME: nginx
LAST DEPLOYED: Thu Jul 31 15:27:24 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the application URL by running this commands:
  minikube service nginx-service --url

```

Check if everything is running
```bash
$ kubectl get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-549fdb7f67-s7xhz   1/1     Running   0          43s
pod/nginx-549fdb7f67-zb8ff   1/1     Running   0          43s

NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP        4h24m
service/nginx-service   NodePort    10.107.150.254   <none>        80:32339/TCP   43s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   2/2     2            2           43s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-549fdb7f67   2         2         2       43s
```

and you can access nginx as earlier

Cleanup:
```bash
helm uninstall nginx
```

###  8) Job and CronJob

Running Nginx using a Job or CronJob in Kubernetes is technically possible, but it's not typical, because Nginx is a long-running web server, and Jobs/CronJobs are designed for short-lived, one-time or periodic tasks (like batch processing, backups, etc.).

Still, for demo purposes or a special use case, here’s how you can run it:

A Job runs to completion, so you need to override the Nginx container to exit after a while — otherwise, the Job will hang.
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: nginx-job
spec:
  ttlSecondsAfterFinished: 30  # auto-delete 30s after success
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        command: ["sh", "-c", "nginx & sleep 10"]
      restartPolicy: Never
  backoffLimit: 1
```

When a Job completes, its Pod moves to the Completed state, but still exists unless manually cleaned up.
Kubernetes has a feature called TTLAfterFinished that lets Jobs self-delete after a specified duration.
```yaml
ttlSecondsAfterFinished: 30  # auto-delete 30s after success
```

If you want to run Nginx periodically, you can use a CronJob, again making sure it doesn't run forever.
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nginx-cronjob
spec:
  schedule: "*/5 * * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: nginx
            image: nginx
            command: ["sh", "-c", "nginx & sleep 10"]
          restartPolicy: OnFailure
```

When a CronJob completes, it creates a Job each time it runs. These jobs and their pods accumulate over time.
```yaml
successfulJobsHistoryLimit: 3 # keeps only the most recent 3 successful jobs
failedJobsHistoryLimit: 1     # keeps only the most recent 1 failed job
```


### Conclusion

Running an NGINX container on a Kubernetes cluster can be achieved through various workload types — from basic Pods to sophisticated tools like Helm charts. Each method serves a unique purpose: Pods and ReplicationControllers help you learn the fundamentals, Deployments and ReplicaSets offer scalability and rollback features, while StatefulSets and DaemonSets cater to specialized workloads. Helm, on the other hand, enables modular, reusable, and version-controlled deployments that scale well in real-world environments.

Although Jobs and CronJobs are not typical choices for running long-lived services like NGINX, they are useful for short-term tasks or demo purposes — provided you handle their cleanup properly using TTLs or history limits.

Understanding when and why to use each method gives you flexibility and control over your Kubernetes workloads, making your architecture both robust and maintainable. Whether you're experimenting locally with Minikube or deploying on a cloud-based EKS cluster, these deployment strategies will enhance your confidence and proficiency in Kubernetes operations.

References:
Medium Blogs by Anvesh Muppeda: https://medium.com/@muppedaanvesh