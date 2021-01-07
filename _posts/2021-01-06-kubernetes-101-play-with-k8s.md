---
title: Kubernetes 101 - Play-with-k8s Labs
author: bigb0ss
date: 2021-01-06 23:53:00 +0800
categories: [Container, Kubernetes, Play-with-k8s Labs]
tags: [kubernetes]
image: /assets/img/post/container/kubernetes/Kubernetes-logo.png
---

# Intro
As new year 2021 is coming along, I was assigned to perform a security assessment focusing on a Kubernetes environment. I had some experience with working Kubernetes and containerized environment before, but honestly, those technologies are like evolving crazy. I believe that Kubernetes is currently the most popular and even being most updated open-source project right now!

I thought it would be a good time to share some resources and techniques that I have learned about Kubernetes (or k8s). I would not go into nitty-gritty about Kubernetes technology itself since there are plethora resources to learn about it like Kube Academy (https://kube.academy/courses).

# Play with Kubernetes

Today, I will demonstrate how to create Kubernetes cluster using free Kubernetes playground tool: Play with Kubernetes (https://labs.play-with-k8s.com/) and deploy simple web application to expose it to the public Internet. 

![image](/assets/img/post/container/kubernetes/play-with-k8s/01.png)

## Login
To login, you need to use either github or docker account. I will use docker account to login. 

![image](/assets/img/post/container/kubernetes/play-with-k8s/02.png)

This will open up another pop-up window to ask for Sign-in.

![image](/assets/img/post/container/kubernetes/play-with-k8s/03.png)

After successful login, click "Start" to start the playground lab.

![image](/assets/img/post/container/kubernetes/play-with-k8s/04.png)

## Add New Instance
The session will be live for 4 hours. After that, it will close session and delete everything. We will click on "+ADD NEW INSTANCE" 3 times to create 3 instances.

![image](/assets/img/post/container/kubernetes/play-with-k8s/05.png)

Node1 (192.168.0.8) - This will be the Kubernetes Mater/Control Plane node.
Node2 (192.168.0.7) - Worker Node1
Node3 (192.168.0.6) - Worker Node2

![image](/assets/img/post/container/kubernetes/play-with-k8s/06.png)

## Initialize Master Node
This lab is awesome that it is already installed with many of the software like kubectl and kubeadm, and it also provide initialization command to setup Kubernetes cluster.  

In the Node1 (192.168.0.8), run the following command:

```console
kubeadm init --apiserver-advertise-address $(hostname -i) --pod-network-cidr 10.5.0.0/16
```

![image](/assets/img/post/container/kubernetes/play-with-k8s/07.png)

At the end of the output, make sure to copy the following commands, especially for the kubeadm join command:

>**Note:** This will be used to join worker nodes to the master node later

```console
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

...snip...

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.8:6443 --token cqrinh.hlgvpsp7ikqa828j \
    --discovery-token-ca-cert-hash sha256:c7b9773c55b7e35a9dda67612bff1129475b36017089c3905383c04d5b43a05d
```

![image](/assets/img/post/container/kubernetes/play-with-k8s/08.png)

## Initialize Networking
When we type `kubectl get nodes` command, we can see that STATUS of the node is NotReady. This is because we have not initialize networking for the cluster. Run the following command to initialize it:

![image](/assets/img/post/container/kubernetes/play-with-k8s/09.png)

```console
kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml
```

![image](/assets/img/post/container/kubernetes/play-with-k8s/10.png)

And if we run the `kubectl get nodes` command again, we can see that the STATUS is now changed to Ready.

![image](/assets/img/post/container/kubernetes/play-with-k8s/11.png)

## Add Worker Nodes
Now, let's add Node2 and Node3 to the master node by using the following command on the each instance:

```console
kubeadm join 192.168.0.8:6443 --token cqrinh.hlgvpsp7ikqa828j \
    --discovery-token-ca-cert-hash sha256:c7b9773c55b7e35a9dda67612bff1129475b36017089c3905383c04d5b43a05d
```

![image](/assets/img/post/container/kubernetes/play-with-k8s/12.png)

To verify, run the `kubectl get nodes -o wide` command on the master node, and we can see that Node2 and Node3 are successfully joined to the master node.

![image](/assets/img/post/container/kubernetes/play-with-k8s/13.png)

## Install Web Application
We will use the following `nginx-app.yaml` file to install a web application (Nginx) to our Kubernetes cluster:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-svc
  labels:
    app: nginx
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

Without going into too much detail, the above file will basically create a LoadBalancer Service to expose the web app using the public IP and deploy 3 replica of Nginx servers.

Let's use the following command to deploy the YAML file: 

```console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/application/nginx-app.yaml
```

![image](/assets/img/post/container/kubernetes/play-with-k8s/14.png)

To verify, run the following command to check the create Nginx servers in the worker nodes:

```console
kubectl get pods -o wide
```

3 replicas of the Nginx servers are successfully created. (2 in Node2 and 1 in Node3)

![image](/assets/img/post/container/kubernetes/play-with-k8s/15.png)

And run the following command to check the LoadBalancer creation:

```console
kubectl get svc -o wide
```

We can see the created the LoadBalancer service, and it is using the port 30736/TCP to expose the Nginx server to the public. 

>**Note:** Kubernetes service will pick random ports between 30000 and 32767 by default unless a port is specified in the deployment script. 

![image](/assets/img/post/container/kubernetes/play-with-k8s/16.png)

## Accessing the Web App
Now, using the public URL for the lab instance + using the port 30736, let's visit the Nginx site via browser.

![image](/assets/img/post/container/kubernetes/play-with-k8s/17.png)

![image](/assets/img/post/container/kubernetes/play-with-k8s/18.png)

# Conclusion
Using Play with Kubernetes lab, I demonstrated the quick setup of Kubernetes cluster + running web application within. This was really simple exercise, and it is not even 1/1000 of what Kubernetes can do. However, I think it is a great start to get your feet wet for Kubernetes at least. Cheers!