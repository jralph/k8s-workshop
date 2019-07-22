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

To access our service from outside of the cluster, we'll need to define a service. Create the below declaration in a `service-yml` file.

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
