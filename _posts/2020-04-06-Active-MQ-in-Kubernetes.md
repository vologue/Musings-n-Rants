---
layout: post
title:  "Deploying Active-MQ in a Kubernetes cluster"
date:   2020-04-06 17:07:40 +0530
categories: kubernetes
---
[Active-MQ][Active-MQ] or apache-amq is a convenient little server which is often used to temporarily store and send messages. In essence, its a message broker which can work with multiple protocols and hence can cater to a larger selection of devices.

It is used to integrate multi-platform applications using the ubiquitous AMQP protocol, exchange messages between web applications using STOMP over web-sockets etc. 



Amazon web services (AWS) offers AmazonMQ as a service. Though this is reliable, it is also very costly.

For example- It costs $0.576 per hour (for an mq.m5.large in the US East (N. Virginia) region)  744(number of hours in a month)*0.576 = $428.54.

Which according to my standards is super expensive. So I decided shift my AmazonMQ db to my Kubernetes cluster. While I was reading up for this I couldn't find many articles and those which I did find were incomplete. So I decided to write a how-to article (This is that article).

In this article, we will be setting up a deployment for Active-MQ. we will also mount the dashboard credentials as a secret (because duh). we will also be creating a service and ingress.

### Step 1: Building the image with docker
For the purpose of this article lets use "[openjdk:8-jre-alpine][openjdk:8-jre-alpine]" as the base image.

- OpenJDK - because active-mq requires java and this saves us the extra step of installing it in ubuntu or alpine base image. 

- 8-jre-alpine - because of its small size. Alpine images are generally very small and bare-bone which makes them light and fast.

My docker file looks a little bit like:

```Dockerfile
FROM openjdk:8-jre-alpine
WORKDIR /home/alpine
RUN apk update && apk add wget
RUN wget -O amq.tar.gz http://apachemirror.wuchna.com//activemq/5.15.10/apache-activemq-5.15.10-bin.tar.gz && tar -xvf amq.tar.gz
EXPOSE 8161 61616 5672 61613 1833
CMD ["/bin/sh","apache-activemq-5.15.10/bin/activemq","console"]
```

Okay let us go through this line by line

- FROM - sets the base image (will pull if not present on building)

- WORKDIR - This sets the working directory in the container. Here the working directory is set to /home/alpine

- RUN - Runs a shell command. Here RUN is used to install wget, which is then used to download AMQ (a tar file) and tar is used to extract it.

- EXPOSE - This allows incoming and outgoing connections to the container at the specified ports. Basically "exposes" the port. Each port specified here is used by a different protocol. A comprehensive list can be found in active-mq's documentation.

- CMD - This directive is used to execute something when the container is initialized. Here it is being used to start the server.

Once the docker file is created, open a terminal and run the following command to build the image.

```bash
docker build -t [name of the image to be created] [path to the dicretory containing the dockerfile]
```

Now to push it to a registry from where Kubernetes can pull the image to create a pod.
```bash
docker push [name of the image]
```

### Step 2: Creating namespaces and secrets

 I think it is a good practice to have every new application in a different namespace as it reduces clutter and often helps in isolating a problem.  To create a namespace just run the following:

```bash
kubectl create namespace active-mq
```
Now that a namespace has been created, we need to add the credentials required to access the dashboard to the Kubernetes cluster as secrets. There are multiple methods to create secrets all of which are documented in k8s's documentation. Here we will be creating it using kubectl.
Create a file "jetty-realm.properties". This is the file which stores user credentials and their roles in active-mq. We are gonna populate this file with custom usernames and passwords and replace the file present by default inside the container.

To the file add the following content:
```
# username: password ,[role-name]
admin_username: admin_pass, admin
guest-user: guest-pass, guest
```
To create the secret run:

```bash
kubectl create secret generic creds --from-file=jetty-realm.properties -n active-mq
```

This will create a secret called "creds" in the namespace active-mq.

### Step 3: Creating the deployment and service objects in Kubernetes

#### Deployment

 This step is pretty straight forward. Only we need to take care while mounting the secret. It is necessary to provide the 'subPath' otherwise the rest of the configuration files will be overwritten and the server won't start. My yaml file looks a little bit like this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
name: active-mq
labels:
   app: active-mq
spec:
replicas: 1
selector:
   matchLabels:
   app: active-mq
template:
   metadata:
   labels:
      app: active-mq
   spec:
   containers:
      - image: [name-of-my-image-from-docker-hub]
      name: active-mq
      imagePullPolicy: Always
      resources:
         requests:
            memory: 500Mi
            cpu: 200m
         limits:
            memory: 1000Mi
            cpu: 400m
      volumeMounts:
      - name: active-creds
        mountPath: /home/alpine/apache-activemq-5.15.10/conf/jetty-realm.properties
        subPath: jetty-realm.properties
   volumes:
   - name: active-creds
     secret:
       secretName: creds
   restartPolicy: Always
```	

This will create a deployment with one pod with custom credentials.

```bash
kubectl create -f deployment.yaml -n active-mq
```

## Service
In this step, we will be creating the yaml for active-mq service and deploying it. In the service.yaml file we must be careful while declaring the ports and only open ports which are being used and name them accordingly. My yaml file looks a little like:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: active-mq
  namespace: active-mq
  labels:
    app: active-mq
spec:
  selector:
    app: active-mq
  ports:
  - name: dashboard
    port: 8161
    targetPort: 8161
    protocol: TCP
  - name: openwire
    port: 61616
    targetPort: 61616
    protocol: TCP
  - name: amqp
    port: 5672
    targetPort: 5672
    protocol: TCP
  - name: stomp
    port: 61613
    targetPort: 61613
    protocol: TCP
  - name: mqtt
    port: 1883
    targetPort: 1883
    protocol: TCP
  type: LoadBalancer
```

To deploy the service to Kubernetes cluster:

```bash
kubectl create -f service.yaml -n active-mq
```

![kubernetes dashboard](/assets/2020-04-06-Active-MQ-in-Kubernetes/k8s-dashboard.png)

This will deploy the service. Since the service is of the type load-balancer, this will expose it to the public at the host-name of the load-balancer set up by your cloud provider. The dashboard and the rest api is exposed on the port 8161. dashboard at '/' and rest api at '/api/message' for more information read the official documentation.

![MQ dashboard](/assets/2020-04-06-Active-MQ-in-Kubernetes/mq-dashboard.png)

If in case you don't want the service to be exposed on a separate load-balancer, then you can set the type to ClusterIP and create an ingress to route the traffic accordingly. However, you can access the dashboard using the port-forward command.

```bash
kubectl port-forward svc/active-mq 8161:8161 -n active-mq
```

Now the dashboard will be available at [http://localhost:8161][localhost].

### Cleanup

To delete the deployment simply delete the namespace. This will remove everything in that namespace (along with Active-mq deployment)

```bash
kubectl delete namespace active-mq
```

[Active-MQ]: https://activemq.apache.org/
[localhost]: http://localhost:8161
[openjdk:8-jre-alpine]: https://hub.docker.com/_/openjdk