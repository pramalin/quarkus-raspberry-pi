  
        Inhabited at some places, deserted at others, 
        depressed at some places, raised at others;
        wherever your (women and) men are good,
        Blessed land, you are good too.

        நாடா கொன்றோ காடா கொன்றோ
        அவலா கொன்றோ மிசையா கொன்றோ
        எவ்வழி நல்லவ ராடவர்
        அவ்வழி நல்லை வாழிய நிலனே.
        Puranaanooru – 187
        [Old Tamil Poetry](https://oldtamilpoetry.com/2016/11/24/puranaanooru-187/)

Table of Contents
=================

   * [Table of Contents](#table-of-contents)
      * [Introduction](#introduction)
         * [Objective](#objective)
         * [Why Raspberry Pi?](#why-raspberry-pi)
      * [Hardware](#hardware)
         * [Power Supply](#power-supply)
      * [Software](#software)
         * [Operating System](#operating-system)
      * [Networking](#networking)
      * [JDK](#jdk)
         * [Maven](#maven)
      * [Docker](#docker)
      * [GrallVM](#grallvm)
      * [Kubernetes](#kubernetes)
         * [Installing k3s](#installing-k3s)
      * [Dashboards](#dashboards)
         * [Installation](#installation)
         * [Dashboard Web access](#dashboard-web-access)
      * [Single Node setup](#single-node-setup)
      * [Function as a Service (FaaS)](#function-as-a-service-faas)
         * [OpenFaas](#openfaas)
      * [Quarkus](#quarkus)
         * [Quick start](#quick-start)
         * [Building docker image](#building-docker-image)
         * [Deploying to Kubernetes](#deploying-to-kubernetes)
         * [Deploying to Raspberry Pi](#deploying-to-raspberry-pi)
            * [Push image to registry](#push-image-to-registry)
            * [Deploying to cluster](#deploying-to-cluster)
            * [kubectl commands examples](#kubectl-commands-examples)
            * [Screenshots](#screenshots)
      * [Useful Links](#useful-links)

## Introduction
[Quarkus](https://quarkus.io/) is a new framework that aims to simplify developing Java applications for container platforms (Docker, Kubernetes, OpenShift, etc). Tools like ['Source to Image'](https://developers.redhat.com/blog/2017/02/23/getting-started-with-openshift-java-s2i/) are already available for this purpose, as seen in this this example: [Deploy a Spring Boot Application to OpenShift](https://www.baeldung.com/spring-boot-deploy-openshift). However Quarkus features fast boot time and small memory usage, making it a better java framework for 'Function as a Service' architecture. Where the functions are instantiated on demand, so fast start up time is a desired feature.

### Objective
This page documents the instructions followed when setting up Raspberry Pi cluster for Kubernetes and running Quarkus examples. The hardware set up and the Operating System installations are based on [Raspberry Pi Dramble](https://www.pidramble.com/). Please refer this site for detailed instructions. We picked [k3s](https://k3s.io/), a lightweight Kubernetes distribution, suitable for machines with small memory capacity. 

### Why Raspberry Pi?
All major cloud system providers support Kubernetes and there are single node Kubernetes VM images suitable to run on developers workstation. However setting Kubernetes up on bare metal servers provides better insight into the operation of multi-node cluster system. Raspberry Pi Single Board Computers are less expensive and a four-node cluster setup with them costs about $300. The book [Kubernetes: Up and Running](https://www.amazon.com/_/dp/1491935677?tag=oreilly20-20) also recommends setting up Raspberry Pi cluster.

Raspberry Pi from model 3 on wards use 64 ARM processors but still use 32 bit Linux for backward compatibility with older models. This can be a limiting factor in some environments. However the same instructions documented here can be adapted for more powerful machines like Intel's [NUC](https://www.amazon.com/dp/B07QH8CG9L/ref=sspa_dk_detail_0?psc=1).

## Hardware
 - Raspberry Pi - 4
 - Micro SD card - 4 
 - 6 inch Ethernet cable - 4
 
### Power Supply
***Option 1***
- USB Power Source
  - USB Power Hub - 1
  - USB to Micro USB cable - 4
  - 4 port Ethernet Switch - 1

***Option 2***
- Power Over Ethernet
  - POE HAT - 4
  - 4 port POE Ethernet Switch - 1

## Software

### Operating System
It is sufficient to use the standard Raspbian (32 bit) Linux distribution to run the JVM version of the container images.
- Flash the OS to SD card
Follow the [official instructions](https://www.raspberrypi.org/documentation/installation/installing-images/README.md) to install Raspbian lite version. 
- enable ssh
Mount the SD card and create a blank file sudo touch /Volumes/boot/ssh

## Networking
Networking setup is the most difficult aspect of the cluster setup. Our requirement is to access each nodes in the cluster individually from the bastion and for each node to have internet access.

Typically the local clusters are demonstrated using a dedicated router for the cluster which then connects to the ISP provided router. For simplicity sake, here we'll use the shared network set up on Ubuntu host, wired to the Ethernet switch of the cluster.

Since we need to login to the individual nodes for administration, it is necessary to assign static IP address to each node. We have used the following IP address for the nodes.

  | ***nodes*** | ***IP address*** |
  |-----------|------------|
  | master0   | 10.0.1.60  |
  | worker1   | 10.0.1.61  |
  | worker2   | 10.0.1.62  |
  | worker3   | 10.0.1.63  |

For each board
- connect via Ethernet port [Link](https://www.raspberrypi.org/magpi/ssh-remote-control-raspberry-pi/)
- login default (user: pi, password: raspberry)
- Edit file: /etc/dhcpcd.conf to set the IP address

```sh
interface eth0
static ip_address=10.0.1.60/24
static ip6_address=fd51:42f8:caae:d92e::ff/64
static routers=10.0.1.1
static domain_name_servers=8.8.8.8
```
- Set hostname in file: /etc/hostname (example)
```sh
master0
```
- Edit file : /etc/hosts to include
```sh
127.0.1.1       master0
```

It is easy to share wireless Internet connection to the wired Ethernet in Ubuntu [Instructions](https://askubuntu.com/questions/3063/share-wireless-connection-with-wired-ethernet-port).

However the default address range (10.42.0.x) of the shared network did not match the static IP addresses selected for the nodes. To change the IP range we need to do the following. Further details are [here](https://askubuntu.com/questions/1062617/cannot-change-address-range-10-42-0-x-in-shared-to-other-computer-method).

```sh
sudo nmcli c modify 'Shared' ipv4.address 10.0.1.1/24
```
This setup will allow us to connect to the individual nodes by IP address and each node will have internet access.

## JDK
Then install the standard JDK for ARM 32 (e.g. jdk-8u212-linux-arm32-vfp-hflt).
```sh
$ sudo apt-get install oracle-java8-jdk
```
### Maven
Install new version of maven in master node, where we will compile our code.
```sh
$ wget https://www-us.apache.org/dist/maven/maven-3/3.6.1/binaries/apache-maven-3.6.1-bin.tar.gz
$ tar zxvf apache-maven-3.6.1-bin.tar.gz
```

## Docker
Docker is also need to be installed in all the nodes. Docker is to verify that the images created can run locally in master before deploying to the cluster. It is also required for the nodes to retrieve the images from repository. In this setup we need to login to docker hub in each node for the deployments to succeed.

```sh
$ curl -sfL https://get.docker.com -o  install-docker.sh
$ ./install-docker.sh
$ sudo docker login -u pramalin
password:
```

## GrallVM
Quarkus support native compilation using GraalVM. However GraalVM distribution is not available for ARM processors yet.


## Kubernetes
We'll use [k3s](https://k3s.io/), a lightweight distribution of Kubernetes targeted for edge devices. This is stripped down version of the official distribution provided by [Rancher Labs](https://rancher.com/). It is interesting to note that there are several certified Kubernetes providers, Google, Amazon, Microsoft, IBM, Red hat, etc.

### Installing k3s
On master node:
- Download k3s install script and install k3s
```sh
$ curl -sfL https://get.k3s.io -o install-k3s.sh
$ ./install-k3s.sh
```
- obtain K3S_TOKEN
```sh
$ sudo cat /var/lib/rancher/k3s/server/node-token
K1067ab3d6b7def59628e00d8498481adfb796365b5b86949250bc5323ad173d040::node:da4b4dcc2bbc88b25f79c7549d93ce36
```
- create start and stop scripts

file: start-k3s.sh 
```sh
sudo k3s server &
```

file: stop-k3s.sh 
```sh
sudo systemctl stop k3s
```
On worker nodes:
- copy /usr/local/bin/k3s from master.
- create start and stop scripts

file: start-k3s-agent.sh 
```sh 
export K3S_URL="https://10.0.1.60:6443"
export K3S_TOKEN=K1067ab3d6b7def59628e00d8498481adfb796365b5b86949250bc5323ad173d040::node:da4b4dcc2bbc88b25f79c7549d93ce36
sudo k3s agent --server ${K3S_URL} --token ${K3S_TOKEN} &
```
file: stop-k3s.sh 
```sh
sudo systemctl stop k3s
```
- ***References***
    - General Kubernetes [documentation](https://kubernetes.io/docs/home/) is the good place to start learning the concepts. 
    
    - [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/docs/) is Google's implementation of Kubernetes on cloud. Their implementation and the Web interface to manage Kubernetes is considered to be the most polished. The documentation has several tutorials for Java examples and offers free credit to test drive their platform.
    
    - [OpenShift](https://learn.openshift.com/) is a Platform as a Service offering from Red Hat, built on top of Kubernetes. This implementation also offers very good Web UI and used by many businesses to implement on-premises Container infrastructure.


## Dashboards
Kubernetes is a command line driven system. The cloud system providers like Google Clouds, AWS provide useful dashboards to manage kubernetes.


[Kubernetes Dashboard](https://github.com/kubernetes/dashboard) can be installed in the Raspberry Pi cluster to serve as UI.

### Installation
[Link](https://mindmelt.nl/mindmelt.nl/2019/04/08/k3s-kubernetes-dashboard-load-balancer/)

> The following procedures are applied in **master** node.

   - setup access control
[Creating Sample user](https://github.com/kubernetes/dashboard/wiki/Creating-sample-user)

***Create yaml files***

file: dashboard-adminuser.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
```
file: adminuser-rbac.yaml
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

***Apply yaml files***
```sh
$ kubectl apply -f dashboard-adminuser.yaml
$ kubectl apply -f adminuser-rbac.yaml
```
***Install dashboard***
   - download dashboard yaml

```sh
curl -sfL https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml > kubernetes-dashboard.yaml
```

file: kubernetes-dashboard.yaml.orig

```yaml
\# ------------------- Dashboard Deployment ------------------- #

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
      - name: kubernetes-dashboard
        image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
        ports:
        - containerPort: 8443
          protocol: TCP

```

file: kubernetes-dashboard.yaml
```yaml
spec:
      containers:
      - name: kubernetes-dashboard
        image: k8s.gcr.io/kubernetes-dashboard-arm:v1.10.1
```

  - edit yaml to use ARM version

  - copy to K3S_LOCATION/manifests in master

```sh
$ sudo cp kubernetes-dashboard.yaml /var/lib/rancher/k3s/server/manifests/
```

***Setup ssh tunnel to proxy***

  -  run kubectl proxy
```sh
$ kubectl proxy
```
- Get  admin token:

```sh
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')

Name:         admin-user-token-89ngz
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 34564883-a1ab-11e9-a0f6-b827ebaf129e

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1062 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTg5bmd6Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIzNDU2NDg4My1hMWFiLTExZTktYTBmNi1iODI3ZWJhZjEyOWUiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.hXGdcBHX4chqnbpiMXQ88062a1lTQwa-dsgsWyLUY7Vgqw_-I5AMdU-V95zBCw3BruvuA8n9JP5S7YTI051sB2dOdzLQuOKtnOajnu21UyPeO3k78DInkjyneinyOhFYlRCPIFEPRR5aebcMUZK82eT4ou0VgWbXOZEfojrvKEDki_uJXDurnc6KLoabdp8P_EoPUTFGF4kax6zQrRytuIhJVj2jTOTQpPM2FEiE_QYWmoivGdjjpIlAuH-XqUSKdp6LeTVdX7Fg_o66flkDeKzJrvPZSp-44VhBHVMYNy2ennOEFQ2hjlVcj-XgCBHk8BSJyGNTFY_cKZCQQ1PDlw


```
### Dashboard Web access

> The following procedure are for **bastion** station.

- start ssh tunnel
```sh 
    ssh -L8001:localhost:8001 pi@10.0.1.60
```
- access [dashboard](http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/)

>   http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

- select Token option and copy paste the token of _admin-user-tonen-xxxx_

## Single Node setup
Single node ready to run VM instances are available for Kubernetes as [minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/) and for OpenShift as [minishift](https://github.com/minishift/minishift).  Which are useful to explore these platforms easily without setting up a cluster. 

## Function as a Service (FaaS)
[AWS Lambda] (https://aws.amazon.com/lambda/) popularized the FaaS model and the recent addition is [Google Cloud Functions](https://cloud.google.com/functions/).

### OpenFaas
[OpenFaas](https://www.openfaas.com/) is a simple platform that can wrap functions written in many languages (Java, JavaScript, Python, Go, R, Shell script, Cobol, etc.) into a Function as a Service modules and deploy to Kubernetes. It offers CLI tool to create, build and deploy the functions and simple Web UI to manage the deployed functions. There is a self-paced [workshop](https://github.com/openfaas/workshop) that can help us understand many practical uses of FaaS  architecture.

## Quarkus
[Quarkus](https://quarkus.io/) is a comprehensive Java framework that compiles Java code into container applications suitable for 'Function as a Service' modules to start up quickly and use less resources.

### Quick start
[Link](https://quarkus.io/guides/getting-started-guide)
***Prerequsites***
1. JDK 1.8+ installed with JAVA_HOME configured appropriately
2. Apache Maven 3.5.3+

***Bootstrapping the project***
```sh
$ mvn io.quarkus:quarkus-maven-plugin:0.18.0:create \
    -DprojectGroupId=org.acme \
    -DprojectArtifactId=getting-started \
    -DclassName="org.acme.getting-started.GreetingResource" \
    -Dpath="/hello"
```

It generates:
- the Maven structure
- an org.acme.getting-started.GreetingResource resource exposed on /hello
- an associated unit test
- a landing page that is accessible on http://localhost:8080 after starting the application
- example Dockerfile files for both native and jvm modes
- the application configuration file

***Running the sample program***
```sh
$ ./mvnw compile quarkus:dev
```
```log
[INFO] Scanning for projects...
[INFO] 
[INFO] ----------------------< org.acme:getting-started >----------------------
[INFO] Building getting-started 1.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ getting-started ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 2 resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ getting-started ---
[INFO] Nothing to compile - all classes are up to date
[INFO] 
[INFO] --- quarkus-maven-plugin:0.17.0:dev (default-cli) @ getting-started ---
Listening for transport dt_socket at address: 5005
2019-07-07 17:30:05,047 INFO  [io.qua.dep.QuarkusAugmentor] (main) Beginning quarkus augmentation
2019-07-07 17:30:12,517 INFO  [io.qua.dep.QuarkusAugmentor] (main) Quarkus augmentation completed in 7470ms
2019-07-07 17:30:15,546 INFO  [io.quarkus] (main) Quarkus 0.17.0 started in 11.381s. Listening on: http://[::]:8080
2019-07-07 17:30:15,588 INFO  [io.quarkus] (main) Installed features: [cdi, resteasy]
```
***Test quarkus web app***
```sh
$ curl http://10.0.1.60:8080/hello
hello
$ curl http://10.0.1.60:8080/hello/greeting/jaxjug
hello jaxjug
```

### Building docker image  
[Link](https://quarkus.io/guides/building-native-image-guide.html)
```
docker build -f src/main/docker/Dockerfile.jvm -t quarkus-getting-started/getting-started .
```
Running locally.
```
docker run -i --rm -p 8080:8080 quarkus-getting-started/getting-started
```

### Deploying to Kubernetes  
[Link](https://quarkus.io/guides/kubernetes-guide.html)

***Prerequisites***
1. having access to a Kubernetes and/or OpenShift cluster. Minikube and Minishift are valid options.
2. being able to package the docker image


### Deploying to Raspberry Pi
By default the container images created by the build system use the JDK for AMD processors as seen in the generated file: src/main/docker/Dockerfile.jvm
This works for typical VMs on most of the developers machines.

```dockerfile
####
# This Dockerfile is used in order to build a container that runs the Quarkus application in JVM mode
#
# Before building the docker image run:
#
# mvn package
#
# Then, build the image with:
#
# docker build -f src/main/docker/Dockerfile.jvm -t quarkus/getting-started-jvm .
#
# Then run the container using:
#
# docker run -i --rm -p 8080:8080 quarkus/getting-started-jvm
#
###
FROM fabric8/java-alpine-openjdk8-jre
ENV JAVA_OPTIONS="-Dquarkus.http.host=0.0.0.0 -Djava.util.logging.manager=org.jboss.logmanager.LogManager"
ENV AB_ENABLED=jmx_exporter
COPY target/lib/* /deployments/lib/
COPY target/*-runner.jar /deployments/app.jar
ENTRYPOINT [ "/deployments/run-java.sh" ]
```

We need to modify the file slightly as shown below.

```dockerfile
FROM hypriot/rpi-java
ENV JAVA_OPTIONS="-Dquarkus.http.host=0.0.0.0 -Djava.util.logging.manager=org.jboss.logmanager.LogManager"
ENV AB_ENABLED=jmx_exporter
RUN mkdir -p /deployments/lib
COPY run-java.sh /deployments
COPY target/lib/* /deployments/lib/
COPY target/*-runner.jar /deployments/app.jar
ENTRYPOINT [ "/deployments/run-java.sh" ]
```

There are two changes to the Dockerfile.jvm
1. use Raspberry Pi JDK image 'hypriot/rpi-java'.
2. copy the run-java.sh which is not available in this image. 

The [run-java.sh](./docs/run-java.sh) file was extracted from the fabric8/java-alpine-openjdk8-jre image. We need to copy this file to the project directory where pom.xml is found. 

After changing the Dockerfile and copying the run-java.sh, the build command generates the docker image that can successfully run in Raspberry Pi.

#### Push image to registry
First, we need to push the Docker image to the image registry of our Kubernetes cluster. Here we have used [docker hub](https://cloud.docker.com/repository/list). This is in some way similar to github where we mnage our source code.
```sh
docker push pramalin/getting-started
```

#### Deploying to cluster
```sh
$ sudo kubectl run quarkus-getting-started --image=pramalin/getting-started:latest --port=8080 --image-pull-policy=IfNotPresent
deployment.apps/quarkus-getting-started created
```
Followed by
```sh
$ sudo kubectl expose deployment quarkus-getting-started --type=NodePort
service/quarkus-getting-started exposed
```
The application is now exposed as an internal service.

Find the assigned port
```sh
$ sudo kubectl get service
NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes                ClusterIP   10.43.0.1       <none>        443/TCP          23h
quarkus-getting-started   NodePort    10.43.146.128   <none>        8080:30966/TCP   4m16s
```

#### kubectl commands examples
```sh
sudo kubectl get deployment
sudo kubectl delete deployment quarkus-quickstart
sudo kubectl get service
sudo kubectl delete service quarkus-quickstart
sudo kubectl get pod
sudo kubectl run quarkus-getting-started --image=pramalin/getting-started:latest --port=8080 --image-pull-policy=IfNotPresent
sudo kubectl expose deployment quarkus-getting-started --type=NodePort
sudo kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
sudo kubectl get service
```

#### Screenshots
![Docker Hub](images/docker-hub.png)
![Quarkus landing page](images/quarkus-landing.png)
![Getting started service](images/getting-started.png)
![Dashboard](images/dashboard.png)

## Useful Links
- [Raspberry Pi Dramble](https://www.pidramble.com/)
- [DrupalCon Seattle 2019: Everything I know about Kubernetes I learned from a cluster of Raspberry Pis](https://www.youtube.com/watch?v=yLzO88h40uw)
- [Will it cluster? k3s on your Raspberry Pi](https://blog.alexellis.io/test-drive-k3s-on-raspberry-pi/)
- [CNCF Presentations](https://github.com/cncf/presentations/tree/master/kubernetes)
- [k3s Lightweight Kubernetes](https://k3s.io/)
- [Quarkus](https://quarkus.io/)
- [kubernetes Documentation](https://kubernetes.io/docs/home/)
- [OpenHab](https://www.openhab.org/)
- [Janakiram](https://www.youtube.com/user/janakirammsv)

- [micronaut](https://micronaut.io/)


