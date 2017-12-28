# k8sIntro

This is a practical example to be used as introductory example for kubernetes.

We will explore a little bit some of the key concepts of kubernetes:

* PODs
* Replication Controllers (Replication Sets now)
* Deployments
* Pacakage Management on kubernetes (Helm)

Kubernetes is an open-source system for automating deployment, scaling, and management of containerized applications.

[Kubernetes Introduction](!https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)

## Starting Minikube

```
minikube start [--vm-driver=xxx]  //by default VBox is used. Check kubectl is configured by defualt to acccess this cluster

kubectl config current-context    // it should return "minikube"

kubectl get nodes                 //It should report a single node NAME=minikube

```

## Create single POD

```
## Sample Pod YAML file used in example (pod.yml):
-----------------------------------------------------------------------
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
spec:
  containers:
  - name: hello-ctr
    image: luismiav/helloworld:latest
    ports:
    - containerPort: 9090
-----------------------------------------------------------------------
```

We will use declarative way always through yaml files. The application we are going to deploy is a very simple
rest service app. When it is called with a GET in this URL "/greeting" it answers with a message and an app id
randomly generetaed for each instance of the application. That is every container will answer with different id.

```
kubectl create -f pod.yml
kubectl get pods  [-o wide]
kubectl describe pods

ping <podIp>      //check POD IP is not reachable from host machine. It is cluster private IP. We cannot access container.

```

## Create a Replication Controllers

```

## Sample Replication Controller YAML file used in example (rc.yml):
-----------------------------------------------------------------------
apiVersion: v1
kind: ReplicationController
metadata:
  name: hello-rc
spec:
  replicas: 3
  selector:
    app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-ctr
        image: luismiav/helloworld:latest
        ports:
        - containerPort: 9090
----------------------------------------------------------------------
```

```
kubectl delete pods hello-pod    // we remove the pod created before
kubectl create -f rc.yml
kubectl get rc -o wide
kubectl describe rc

kubectl get pods -o wide   // see the pods running
```

We can now change the replication controller yaml file and establish 4 replicas and apply the changes

```
kubectl apply -f rc.yml
kubectl get pods -o wide   // check that a new replica is started.
```

## Create a Service

```
### The following is the "svc.yml" file used:
----------------------------------------------------------------------

apiVersion: v1
kind: Service
metadata:
  name: hello-svc
  labels:
    app: hello-world
spec:
  type: NodePort
  ports:
  - port: 9090
    nodePort: 30001
    protocol: TCP
  selector:
    app: hello-world
----------------------------------------------------------------------

```

```

kubectl create -f svc.yml
kubectl get svc -o wide
kubectl describe svc hello-svc

minikube ip   // get minikube-ip. This is the IP where we can access the node in the minikube VM

curl http://minikube-ip:30001/greeting  //call it several times to see random load balancer

```
Now with the service we can access all the PODs that are behind this service. The three of them randomly.
Check it sending several curls.

With this our small application composed by three PODs, to provide high availability is fully functional
and we can access it from outside world.

But next issue is What if I need to evolve it?. How can I upgrade it. Let's see wht deployments add to this

## Deployments  (Upgrades & Rollbacks)

```
################# Simple deployment YAML file used (deployment.yml)
----------------------------------------------------------------------
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-pod
        image: luismiav/helloworld:latest
        ports:
        - containerPort: 9090
----------------------------------------------------------------------
```

```

kubectl create -f deployment.yml
kubectl get deployment -o wide
kubectl describe deployment hello-deploy  // Note StrategyType

kubectl get rs - o wide
kubectl describe rs

```
Let's make an upgrade of the application to a new release. New message will be shown.
We can just change the deployment file to apply an upgrade. This can use version control in reality
The upgrade is highly configurable.
We are going to use RollingUpdate strategy and

```
################# Simple deployment YAML file used for upgrading app (upgrade.yml)
----------------------------------------------------------------------
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  replicas: 3
  minReadySeconds: 10  //wait for 10 secs after a new POD is up before starting with next
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  // One POD new at a time
      maxSurge: 1        // Never more than one extra POD
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-pod
        image: luismiav/helloworld:edge
        ports:
        - containerPort: 9090
----------------------------------------------------------------------
```

```
kubectl apply -f upgrade.yml --record   // record will

kubectl rollout status deployments hello-deploy

kubectl get deploy hello-deploy

kubectl rollout history deployments hello-deploy

kubectl get rs

kubectl describe deploy hello-deploy

kubectl rollout undo deployment hello-deploy --to-revision=1

kubectl get deploy

kubectl rollout status deployments hello-deploy

```

## Helm. Package Manager for k8s

The first thing to do is to install the helm server (Tiller) into the cluster (minikube)

```

helm init // this will install Tiller in the minikube cluster
helm version   // check that you get version of client and server. It can take some small time the server to boot up

helm create helloworld // this will create a package structure that we can customize

```

We need to adapt the values.yaml file to the need of our application in order to get same behavior we used to
have without helm.
There are many ways and this is one that should work. We are using only few values to get the behavior.

```
----------------------------------------------------------------------
# Default values for helloworld.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.
replicaCount: 3
image:
  repository: luismiav/helloworld
  tag: latest
  pullPolicy: IfNotPresent
service:
  name: hello-svc
  type: NodePort
  externalPort: 30001
  internalPort: 9090
ingress:
  enabled: false
  # Used to create an Ingress record.
  hosts:
    - chart-example.local
  annotations:
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  tls:
    # Secrets must be manually created in the namespace.
    # - secretName: chart-example-tls
    #   hosts:
    #     - chart-example.local
resources: {}
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  # limits:
  #  cpu: 100m
  #  memory: 128Mi
  # requests:
  #  cpu: 100m
  #  memory: 128Mi
----------------------------------------------------------------------
 ```

With this values helm will be filling the templates that have already created and it will make the kubernetes
deployment and the service for connecting our application. The connection will be made with a mapped port
but the notes.txt will give us clear instructions on how to access the application.

```

helm install --dry-run --debug ./helloworld

```

This will print out the real manifest files helm is going to use for kubernetes deployment and service
In the deployment.yaml you can check that by default livenessProbe and readinessProbe are set.
We will change this in the /templates/deployment.yaml to comment all these lines since this will not
work with our simple app and it will make kubernetes restart the POD and container inside ciclycally.

After this change we can just install the package giving a name for the release

```

helm install --name hellorelease ./helloworld

kubectl get deployments -o wide
kubectl get pods -o wide

```

You will see some information about service, PODs and NOTES on how to access the app URL

Try to do same curl as before but with this URL, and you will get same

Now we are going to upgrade in the same way the application as before:

We can copy the helloworld content into another folder helloworld-upgrade and change this parts:

```

Chart.yaml
----------------------------------------------------------------------
apiVersion: v1
description: A Helm chart for Kubernetes
name: helloworld
version: 0.2.0                           // we just change the release version
----------------------------------------------------------------------

values.yaml
----------------------------------------------------------------------

# Default values for helloworld.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.
replicaCount: 3
image:
  repository: luismiav/helloworld
  tag: edge                         // we just change the image tag to be used
  pullPolicy: IfNotPresent
service:
  name: hello-svc
  type: NodePort
  externalPort: 30001
  internalPort: 9090

....

```

That's all. We just leave default strategy for upgrade and values.

```

helm ls --all  // if we do not remmeber releases we are handling

helm upgrade  hellorelease ./helloworld-release

kubectl get pods -o wide

helm history hellorelease       // you can check the historical versioning of releases

```

Try again the curl and check the app has been upgraded

Rolling back to version 1 of the app is as easy as this

```

helm rollback hellorelease 1

```
