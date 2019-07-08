## Quarkus on Raspberry PI nodes

* [Quarkus on Raspberry PI nodes](#quarkus-on-raspberry-pi-nodes)
   * [Introduction](#introduction)
   * [Hardware](#hardware)
   * [Power Supply](#power-supply)
   * [Software](#software)
      * [Operating System](#operating-system)
      * [Networking](#networking)
      * [Kubernetes](#kubernetes)
      * [Quarkus](#quarkus)
      * [Examples](#examples)
   * [Other Cluster setups](#other-cluster-setups)
   * [References](#references)

### Introduction
[Quarkus](https://quarkus.io/) is a new framework aims to simplify developing Java applications for container platforms (Docker, Kubernetes, OpenShift, etc). Tools like ['Source to Image'](https://developers.redhat.com/blog/2017/02/23/getting-started-with-openshift-java-s2i/) are already available for this purpose, as seen in this this example: [Deploy a Spring Boot Application to OpenShift](https://www.baeldung.com/spring-boot-deploy-openshift). Quarkus features fast boot time and small memory usage, making it better framework to develop Java applications for 'Function as a Service' architecture where the functions are instantiated on demand, unlike long running applications that may tollerate slow start up.

#### Objective
This page documents the instructions followed when setting up Rapberry Pi cluster for Kubernetes and running Quarkus examples. Overall the hardware set up is based on [Raspberry Pi Dramble](https://www.pidramble.com/) and Kubernetes set up is based on [k3s on Raspberry Pi](https://blog.alexellis.io/test-drive-k3s-on-raspberry-pi/). Please use them for detailed instructions.

### Why Raspberry Pi?
All major cloud system providers support Kubernetes and there are single node Kubernetes VM images suitable to run on developers workstation. However setting Kubernetes up on bare metal servers provides better insight into the operation of multi-node cluster system. Raspberry Pi Single Board Computers are less expensive and acluster with one master node and 3 worker nodes can be setup spending less than 300 US dollers. The book [Kubernetes: Up and Running](https://www.amazon.com/_/dp/1491935677?tag=oreilly20-20) also recomends setting up Raspberry Pi cluster.

Raspberry Pis from model 3 onwards use 64 ARM processors but still use 32 bit Linux for backward compatibility with older models. When its hardware becomes a limiting factor for their needs, many developers use stack of Intel's [NUC](https://www.amazon.com/dp/B07QH8CG9L/ref=sspa_dk_detail_0?psc=1) or similar hardware.

### Hardware
 - Rabpberry Pi - 4
 - Micro SD card - 4 
 - 6 inch Ethernet cable - 4
 
### Power Supply
__Option 1__
- USB Power Source
  - USB Power Hub - 1
  - USB to Micro USB cable - 4
  - 4 port Ethernet Switch - 1

__Option 2__
- Power Over Ethernet (Option 2)
  - 4 port POE Ethernet Switch - 1

### Software

#### Operating System
It is sufficient to use the standard raspbian (32 bit) Linux distribution to run the JVM version of the container images.

#### JDK
 We just need to install the standard JDK for ARM 32 (e.g. jdk-8u212-linux-arm32-vfp-hflt).

#### GrallVM
Quarkus support native compilation using GraalVM. However GraalVM distribution is not available for ARM processers yet.


#### Networking
Typically the local clusters are demonstrated using a dedicated router for the cluster which then connects to the ISP provided router. For simplicity sake, here we'll use the shared network set up on Ubuntu host, wired to the Ethernet switch of the cluster.

Since we need to login to the individual nodes for administration, it is necessary to assign static IP address to each node. We have used the following IP address for the nodes.

  | **nodes** | **IP address** |
  |-----------|------------|
  | master0   | 10.0.1.60  |
  | worker1   | 10.0.1.61  |
  | worker2   | 10.0.1.62  |
  | worker3   | 10.0.1.63  |


It is easy to share Wifi Internet connection to the wired Ethernet in Ubuntu [Instructions](https://askubuntu.com/questions/3063/share-wireless-connection-with-wired-ethernet-port).


However the default address range (10.42.0.x) of the shared network did not match the static IP addresses selected for the nodes. To change the IP range we need to do the following. Further details are [here](https://askubuntu.com/questions/1062617/cannot-change-address-range-10-42-0-x-in-shared-to-other-computer-method).

```sh
sudo nmcli c modify 'Shared' ipv4.address 10.0.1.1/24
```

#### Kubernetes
We'll use [k3s](https://k3s.io/), a lightweight distribution of Kubernetes targeted for edge devices. This is stripped down version of the official distribution provided by [Rancher Labs](https://rancher.com/). It is interesting to note that there are several certified Kubernetes providers, Google, Amazon, Microsoft, IBM, Red hat, etc.

**References**
- General Kubernetes [documentation](https://kubernetes.io/docs/home/) is the good place to start learning the concepts. 

- [Goole Kubernetes Engine](https://cloud.google.com/kubernetes-engine/docs/) is Goole's implementation of Kubernetes on cloud. Their implementation and the Web interface to manage Kubernetes is considered to be the most polished. The documentation has several tutorials for Java examples and offers free credit to test drive their platform.

- [OpenShift](https://learn.openshift.com/) is a Platform as a Service offering from Red Hat, built on top of Kubernetes. This implementation also offers very good Web UI and used by many businesses to implement on-premises Container infrastructure.


#### Dashboards
Kubernetes is a command line driven system. The cloud system providers like Google Clouds, AWS provide useful dashboards to manage kubernetes.



[Kubernetes Dashboard](https://github.com/kubernetes/dashboard) can be installed in the Raspberry Pi cluster to serve as UI.

#### Single Node setup
Single node ready to run VM instances are available for Kubernetes as [minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/) and for OpenShift as [minishift](https://github.com/minishift/minishift).  Which are useful to explore these platforms easily before trying to setup the cluster. 

### Function as a Service (FaaS)
AWS Lambda populerized the FaaS model

#### OpenFaas
[OpenFaas](https://www.openfaas.com/) is a simple platform that can wrap functions written in many languages (Java, JavaScript, Python, Go, R, Shell script, Cobol, etc) into a Function as a Service modules and deploy to Kubernetes. It offers CLI tool to create, build and deploy the functions and simple Web UI to manage the deployed functions. There is a self-pased [workshop](https://github.com/openfaas/workshop) that can help us understand many practical uses of FaaS  architecture.

### Quarkus
[Quarkus](https://quarkus.io/) is a comprehensive Java framework that compiles Java code into container applications suitable for 'Function as a Service' modules to start up quickly and use less resources.

#### Quick start
[Link](https://quarkus.io/guides/getting-started-guide)
**Prerequsites**
1. JDK 1.8+ installed with JAVA_HOME configured appropriately
2. Apache Maven 3.5.3+

**Bootstrapping the project**
```sh
$ mvn io.quarkus:quarkus-maven-plugin:0.18.0:create \
    -DprojectGroupId=org.acme \
    -DprojectArtifactId=getting-started \
    -DclassName="org.acme.quickstart.GreetingResource" \
    -Dpath="/hello"
```

It generates:
- the Maven structure
- an org.acme.quickstart.GreetingResource resource exposed on /hello
- an associated unit test
- a landing page that is accessible on http://localhost:8080 after starting the application
- example Dockerfile files for both native and jvm modes
- the application configuration file

**Running the sample program***
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
$ curl http://localhost:8080/hello
hello

#### Building docker image  
[Link](https://quarkus.io/guides/building-native-image-guide.html)
```
docker build -f src/main/docker/Dockerfile.jvm -t quarkus-quickstart/getting-started .
```
Running locally.
```
docker run -i --rm -p 8080:8080 quarkus-quickstart/getting-started
```

#### Deploying to Kubernetes  
[Link](https://quarkus.io/guides/kubernetes-guide.html)

**Prerequisites**
1. having access to a Kubernetes and/or OpenShift cluster. Minikube and Minishift are valid options.
2. being able to package the docker image from the native application guide



#### Deploying to Raspberry Pi
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
2. copy the file run-java.sh which is not available in this image. 

The [run-java.sh](./docs/run-java.sh) file was extracted from the fabric8/java-alpine-openjdk8-jre image. We need to copy this file to the project directory where pom.xml is found. 

After changing the Dockerfile and copying the run-java.sh, the build command generates the docker image that can successfully run in Raspberry Pi.


### References
- [Raspberry Pi Dramble](https://www.pidramble.com/)
- [CNCF Presentations](https://github.com/cncf/presentations/tree/master/kubernetes)
- [k3s Lightweight Kubernetes](https://k3s.io/)
- [Quarkus](https://quarkus.io/)
- [kubernetes Documentation](https://kubernetes.io/docs/home/)
- [OpenHab](https://www.openhab.org/)
- [Janakiram](https://www.youtube.com/user/janakirammsv)
