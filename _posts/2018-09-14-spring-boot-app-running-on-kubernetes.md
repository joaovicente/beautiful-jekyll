---
layout: post
published: false
title: Spring boot app running on Kubernetes
---
[Kubernetes tutorial](https://kubernetes.io/docs/tutorials/kubernetes-basics/)

## Create a local Docker registry

```
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

> See [this article](https://blog.hasura.io/sharing-a-local-registry-for-minikube-37c7240d0615) for more context on running a local registry

## Change the `pom.xml` to push to local Docker registry

```
<properties>
    <docker.image.prefix>localhost:5050</docker.image.prefix>
</properties>
```

## Push the docker image into the local Docker repository
```
//FIXME (triggering an error)
$ mvn docker:build -DpushImage -Ddocker.repository=localhost:5000
```

## Install Kompose

[Kompose](https://github.com/kubernetes/kompose/blob/master/docs/getting-started.md) helps transitioning from docker-compose to kubernetes, by converting `docker-compose.yaml` files to equivalent kubernetes deployment and service files

```
$ curl -L https://github.com/kubernetes/kompose/releases/download/v1.16.0/kompose-linux-amd64 -o kompose
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   609    0   609    0     0     96      0 --:--:--  0:00:06 --:--:--   161
100 45.6M  100 45.6M    0     0   443k      0  0:01:45  0:01:45 --:--:--  500k

$ chmod +x kompose
$ sudo mv ./kompose /usr/local/bin/kompose
```

## Change the `docker-compose.yaml` to point to localhost:5000

```
version: '2'
services:
  demo:
    image: localhost:5000/demo:latest
    ports:
      - "8080:8080"
```

## Convert the docker-compose into pod and service resource descriptors

```
$ kompose convert
INFO Kubernetes file "demo-service.yaml" created  
INFO Kubernetes file "demo-deployment.yaml" created
```

## Check the state of you Kubernetes cluster

```
$ kubectl get nodes
$ kubectl get pods
```

## Run the deployment
```
$ kubectl create -f demo-deployment.yaml"
```

## Run the service
```
$ kubectl create -f demo-service.yaml"
```

## Test the service
```
$ kubectl get services
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
demo         ClusterIP   10.106.18.44   <none>        8080/TCP   12s
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP    2d
$ http localhost:8080
// FIXME: cannot connect via   
```


## The cheat sheet
https://kubernetes.io/docs/reference/kubectl/cheatsheet/



---


kubectl get deployments
kubectl run kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1 --port=8080
kubectl get deployments





---

Useful commands
kubectl version
kubectl get nodes

---

## troubleshooting localhost:5000 registry 


> Docker Registry REST API: https://docs.docker.com/registry/spec/api/

```

http get http://localhost:5000/v2/
http get http://localhost:5000/v2/_catalog
```


To pull an image manifest: `http get http://localhost:5000/v2/{repository/manifests/tag`
```
## e.g.
http get http://localhost:5000/v2/documentation/manifests/latest
http http://localhost:5000/v2/demo/manifests/latest 'Accept: application/vnd.docker.distribution.manifest.v2+json'
```



docker run -d -p 5000:5000 -e "REGISTRY_STORAGE_DELETE_ENABLED=true" --restart=always --name registry registry:2
