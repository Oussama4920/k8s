# Project Introduction

In this project, the goal is to deploy a webapp on NGINX linked to an external database “mongoDB” deployed on another node on K8S using minikube as the k8s cluster and other components (Secret, Deployment, service and configMap. 

We’ll discuss the architecture and the choice of the several components below.

![image](https://github.com/user-attachments/assets/5a337ca5-de07-4c47-9088-5c1afbe95e23)

## Service (SVC)

- A k8s component that provides a pod a **permenant and stable IP address**/DNS name
- This IP address is permenant even after an update, a scaling or a failure
    - Types of services :
        - **ClusterIP (default)**: Exposes the Service on an **internal** IP address only accessible within the Kubernetes cluster.
        - **NodePort**: Exposes the Service on each node’s IP address at a static port. This makes it accessible **externally, but on a fixed port number**
        - **LoadBalancer**: Exposes the Service **externally using a cloud provider’s load balance**r. This is often used in public cloud environments.
        - **ExternalName**: Maps a Service to a DNS name outside of the Kubernetes cluster.
        

In this project we will be using `ClusterIP`inside of minikube cluster for our DB service (we don’t want external access to our DB) and `NodePort`for our webapp service to access externally.

## ConfigMap (CM)

Our DB service may change name due to an update, scaling … etc, we should then change our webapp configuration file also to access DB everytime than happens and rebuild our webapp image ... This is not ideal.

That’s why we’ll use configMap component. It is an API object used to store non-confidential configuration data in key-value pairs 

It allows you to separate **external configuration from application code**, making it easier to update and manage configuration data without rebuilding or redeploying your containers.

In this project we’ll store the key value pair `mongo-url:mongo-service`

## Secret

API object that is **used to store and manage sensitive information**, such as passwords, tokens, SSH keys, or any other confidential data. 

Secrets help ensure that sensitive data does not get hardcoded or exposed in application code or container images.

As all DBs, mongoDB requires athentification also, so in this project we will be storing our DB creds in this .yaml file coded in base64 to avoid hardcoding them or put them in a configmap in a plain text.

## Deployment

- It is a the bleuprint for pods management
- Used to manage **stateless** applications in Kubernetes, offering features such as rolling updates, rollbacks, and scaling
- Ensure that a specified number of identical Pods (replicas) are running at any given time, and that updates to these Pods are performed in a controlled manner

For DBs we dont use Deployments since a DB is a stateful componenet (has a state “its data”) so we need a component that will take this into consideration and sync DBs for all replicas.

For simplification purposes, in this project we will use a deployment component for mongo with only one replica so we don't need synchronization … In a real Prod env we will use StatefulSets (sts).

## Minikube

Lightweight Kubernetes implementation that allows you to run a single-node Kubernetes cluster locally on your personal machine.

 It is designed for development, testing, and learning purposes, providing an easy way to get started with Kubernetes without needing a full cloud infrastructure. 

# Steps to create and deploy the target architecture above

- Install Docker and make sure the docker driver is running
- Install minikube (minikube.exe)
- Start minikube

```bash
minikube start
```

- Create a `mongo-config.yaml` file for the configmap component

```yaml
apiVersion: v1
kind: ConfigMap #Kind of the component
metadata:
  name: mongo-config #name of the component
data:   #the data we need for external config, here it's the name of the DB service
  mongo-url: mongo-service
```

- Create a `mongo-secret.yaml` file for the secret component

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret 
type: Opaque   #type of the secret, using the default type
data:
  mongo-user: bW9uZ291c2Vy         # "myuser" in base 64 to not write it in plain text
  mongo-password: bW9uZ29wYXNzd29yZA==
```

- Create a `mongo.yaml` for the Deployment component and the service component (commonly done on the same file since every Deployment needs a service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-deployment
  labels:              # Optional here but preffered to use to manage and identify ressources in k8s more effeciently
    app: mongo
spec:
  replicas: 1    #number of the replicas, here pods, we said above, we only using one for this demo
  selector:            #the spec part of the component has a selector field to point on the ressources targeted by this configuration, in this case, we'll target the pods under the label of "mongo"
    matchLabels:
      app: mongo
  template:           #The blueprint of our pods (how are they cretaed, managed ...)
    metadata: 
      labels:
        app: mongo  #label of our pods (Required here)
    spec:
      containers:
      - name: mongodb
        image: mongo:5.0 #official image of mongo software from DockerHub https://hub.docker.com/_/mongo
        ports:
	        - containerPort: 27017  #specified in the image documentation
        env:   #Environement variables that we'll be using for our pods, here since it is a DB, it requires username and password so the webapp can access the DB
        - name: MONGO_INITDB_ROOT_USERNAME #name of the env variable, mentionned in the docker image documentation
          valueFrom: #help us retreive the value from secret component (we see here the added value of referencing and using secrets)
            secretKeyRef: 
              name: mongo-secret
              key: mongo-user
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: mongo-password
---  #in YAML, we can use this seperation instead of two files, makes sense here since everydeployment needs a service so better group them together
apiVersion: v1
kind: Service
metadata:
  name: mongo-service
spec:
  selector:
    app: mongo
  ports:
    - protocol: TCP
      port: 27017  # the port where we will receive the request, it can be any available port (we use the sameas the target port here)
      targetPort: 27017 # the port where to route the request, it should be the port where the mongo depoyment is listening
```

- Create another Deployment for the webapp `webapp.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  labels:
    app: webapp
spec:
  replicas: 1 
  selector:
    matchLabels:
      app: webapp
  template:
    metadata: 
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nanajanashia/k8s-demo-app:v1.0 #the docker image of our webapp, I used an already made simple webapp on NGINX publically available on DockerHub
        ports:
        - containerPort: 3000 #mentionned in the image documentation
        env:
        - name: USER_NAME
          valueFrom:
            secretKeyRef: 
              name: mongo-secret
              key: mongo-user
        - name: USER_PWD
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: mongo-password
        - name: DB_URL
          valueFrom:
            configMapKeyRef: #we can use this to retreive data from the configMap as we did earlier with secret component
              name: mongo-config #name of the configMap
              key: mongo-url #the key of the data we want
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort #we will be using this type of port instead of internal port by default (ClusterIP) so we can access our webapp from the browser
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 3000 #where the service listen
      targetPort: 3000 #where the request should be routed, here our webapp that is listening on port 3000
      nodePort: 30100 #field we should add since the type is nodePort, this port is used to access the webapp from outside the cluster (http://node-ip:nodePort)
```

- Lunch the components (note: mongo.yaml should start before the webapp)

```bash
kubectl apply -f mongo-config.yaml
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo.yaml 
kubectl apply -f webapp.yaml
```

- Get the IP to target the minikube cluster from the browser (here it’s [http://192.168.49.2:30100](http://192.168.49.2:30100/))

```bash
minikube ip
```

- Check the browser

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/b2e848df-9ad3-48fe-82e5-a2ae2f74624c/fcb64891-1b96-4cc8-bb3a-417b882da631/image.png)

**Note**: here we are using a tunnel from the IP to target the cluster to the [localhost](http://localhost), since i couldn’t access from the [http://192.168.49.2:30100](http://192.168.49.2:30100/), this may be due to the port is busy.

```bash
minikube service webapp-service
|-----------|----------------|-------------|---------------------------|
| NAMESPACE |      NAME      | TARGET PORT |            URL            |
|-----------|----------------|-------------|---------------------------|
| default   | webapp-service |        3000 | http://192.168.49.2:30100 |
|-----------|----------------|-------------|---------------------------|
* Starting tunnel for service webapp-service.
|-----------|----------------|-------------|------------------------|
| NAMESPACE |      NAME      | TARGET PORT |          URL           |
|-----------|----------------|-------------|------------------------|
| default   | webapp-service |             | http://127.0.0.1:54571 |
|-----------|----------------|-------------|------------------------|
```
