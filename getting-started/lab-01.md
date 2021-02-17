# Getting Started

## Prerequisites
- Have a cluster running and `kubectl` configured to talk to that cluster.

## Lab

### Pods

First define a single pod. Create a file called `pod.yml` locally and start this app from this declaration.

The below declaration will start a pod called `lobsters`, that runs a single container using the image `gcr.io/google-samples/lobsters:1.0`. It gives the pod a name of `lobsters` and a label of `app.kubernetes.io/name: lobsters`. It names the container `lobsters` and specifies that the container exposes a port of `3000` and to call this port `web`.

```yml
apiVersion: v1
kind: Pod
metadata:
  name: lobsters
  labels:
    app.kubernetes.io/name: lobsters
spec:
  containers:
  - image: gcr.io/google-samples/lobsters:1.0
    name: lobsters
    ports:
    - containerPort: 3000
      name: web
```

Start the pod.

```sh
kubectl apply -f ./pod.yml
```

Check the pod has been created.

```sh
kubectl get pods
```

```txt
NAME       READY     STATUS    RESTARTS   AGE
lobsters   1/1       Running   0          1m
```

Delete the pod.

```sh
kubectl delete pod lobsters
```

The pod is gone forever.

### Service

To access our service from outside of the cluster, we'll need to define a service. Create the below declaration in a `service.yml` file.

The declaration below defines a service that will route traffic to any pod with the label `app.kubernetes.io/name: lobsters` we created in the pod above. This service routes requests to the port `web`, that we specified in the container spec in our pod declaration. The `type: NodePort` line allows traffic on a particular port of each node to be routed to the service. As we are running locally, this will be `localhost` followed by a random port (for example, `localhost:30821`).

```yml
apiVersion: v1
kind: Service
metadata:
  name: lobsters
  labels:
    app.kubernetes.io/name: lobsters
spec:
  ports:
    - port: 80
      targetPort: web
      name: web
  selector:
    app.kubernetes.io/name: lobsters
  type: NodePort
```

Create the service and pod.

```sh
kubectl apply -f ./service.yml,./pod.yml
```

Check the service's node port:

```sh
kubectl get service lobsters
```

```txt
NAME       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
lobsters   NodePort   10.105.84.179   <none>        80:30957/TCP   79m
```

The part we are looking for is the `PORT(S)` column. This is telling us that port `80` of our service is exposed as port `30957` on the local node.

Check the service is working by visiting the NodeIP `localhost:30957`.

Delete the service and pod by their shared labels.

```sh
kubectl delete pod,service -l app.kubernetes.io/name=lobsters
```

### Deployments

When we delete the above pod, it stays deleted. Your pod can also disappear if your cluster node fails, or if the app crashes and can't be restarted. The Kubernetes solution to this is a Deployment. A deployment will create a Replica Set that ensures a pod or number of pods is always running somewhere in the cluster. In fact, it is almost never appropriate to create individual pods as we did above.

In the below declaration we create a deployment of our lobster app. We give it the usual labels so we know this deployment is part of the lobster app. We define `relicas: 1` to tell the deployment that there should always be 1 copy of the pod running at all times. We specify a selector to tell the deployment how to find which pods to manage. We define a template to tell the deployment how to create new pods when needed. This is very similar to the `spec` section our pod declaration above.

Create a file called `deployment.yml` with the below deployment declaration.

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lobsters
  labels:
    app.kubernetes.io/name: lobsters
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: lobsters
  template:
    metadata:
      labels:
        app.kubernetes.io/name: lobsters
    spec:
      containers:
      - name: lobsters
        image: gcr.io/google-samples/lobsters:1.0
        ports:
        - containerPort: 3000
          name: web
```

```sh
kubectl apply -f ./service.yml,./deployment.yml
```

Check the deployment status:

```sh
kubectl get deployment lobsters
```

```txt
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
lobsters   1/1     1            1           10s
```

Check the service's node port again:


```sh
kubectl get service lobsters
```

```txt
NAME       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
lobsters   NodePort   10.105.84.179   <none>        80:30957/TCP   79m
```

Check the service is working by visiting the NodeIP `localhost:30957`.

Now, look at the pod.

```sh
kubectl get pods
```

You will see there is now a hash/code after the pod name. This is because it was created by our deployment.

Try deleting a pod.

```sh
kubectl delete pod lobsters-jf0xs
```

Let's look at our pods again:

```sh
kubectl get pods
```

You can see we have a pod with a new name starting up, or already running.

Scaling this is easy:

```sh
kubectl scale --replicas=5 deployment lobsters
```

We can also edit the `deployment.yml` file and set `replicas: 5`, then reapply the file. This would give the same result.

Check the deployment status:

```sh
kubectl get deployment lobsters
```

```txt
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
lobsters   2/5     1            1           5m
```

Check the pods:

```sh
kubectl get pods
```

You should now see a total of 5 pods.

Let's update the image to 2.0!

In your `deployment.yml` file, change the container image to `gcr.io/google-samples/lobsters:2.0`.

Let's apply our updated deployment.

```sh
kubectl apply -f deployment.yml
```

Now if we look at the pods, we should gradually see the new pods start to appear, and the old ones disappear. This is achieved by the deployment creating a new replica set and gradually scaling it up, while gradually scaling down and deleting the old replica set.

We can see this.

```sh
kubectl get replicaset
```

On running this command again, you should see the number of replicas gradually change, if the rollout has not already completed!

Delete the deployment and service:

```sh
kubectl delete deployment,service -l app.kubernetes.io/name=lobsters
```

### Ingress

Above, we've been using services with a type of `NodePort` to gain access to our pods/containers. In the real world, this might not be all that suitable as you may have load balancers and dns to contend with.

This is where kubernetes introduces `Ingress`.

An ingress is a configuration of routing of how an ingress controller should route traffic to a service.

Running an ingress controller locally is out of the scope of this guide, but below is an example of how to connect our lobster service to an ingress controller using an `Ingress`.

The below declaration will setup an ingress rule that any host matching `lobsters.example.com` with the path starting with `/` should be routed to our service at the port `web`.

```yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: lobsters
  labels:
    app.kubernetes.io/name: lobsters
spec:
  tls:
  - secretName: lobsters.example.com
  rules:
  - host: lobsters.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: lobsters
          servicePort: web
```

## Cleanup

Delete everything we created in this lab.

```sh
kubectl delete pod,deployment,service -l app.kubernetes.io/name=lobsters
```
