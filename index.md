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
These are the notes kept when setting up Rapberry Pi cluster for Kubernetes and running Quarkus. Overall the hardware setup is based on [Raspberry Pi Dramble](https://www.pidramble.com/) and Kubernetes setup is done based on [OpenFaas](https://blog.alexellis.io/test-drive-k3s-on-raspberry-pi/). Please follow them for detailed instructions.

Typically the local clusters are demonstrated using a dedicated router. Here we are using shared network setup on Ubuntu host, wired to the Ethernet switch of the cluster.

### Hardware
 - Rabpberry Pi - 4
 - Micro SD card - 4 
 - 6 inch Ethernet cable - 4
 
### Power Supply
- USB Power Source (Option 1)
  - USB Power Hub - 1
  - USB to Micro USB cable - 4
  - 4 port Ethernet Switch - 1


- Power Over Ethernet (Option 2)
  - 4 port POE Ethernet Switch - 1

### Software

#### Operating System
- Standard raspbian (32 bit)
- Etcher


Quarkus support native compilation using GraalVM. However GraalVM is available for ARM processers that are used in Raspberry yet. One may try installing 64 bit Linux for ARM in the Raspberry Pi and compile GraalVM from source.


#### Networking
We have used the following IP address for the nodes.

  | **nodes** | **IP address** |
  |-----------|------------|
  | master0   | 10.0.1.60  |
  | worker1   | 10.0.1.61  |
  | worker2   | 10.0.1.62  |
  | worker3   | 10.0.1.63  |


It is easy to setup shared Network connection to wired Ethernet in Ubuntu [Instructions](https://askubuntu.com/questions/3063/share-wireless-connection-with-wired-ethernet-port).
However the address range used by default (10.42.0.x) did not match the static IP addresses selected for the nodes.
To change the IP range do the following. Further details are [here](https://askubuntu.com/questions/1062617/cannot-change-address-range-10-42-0-x-in-shared-to-other-computer-method).

```sh
sudo nmcli c modify 'Shared' ipv4.address 10.0.1.1/24
```

#### Kubernetes
We are using [k3s](https://k3s.io/), a lightweight distribution of Kubernetes targeted for edge devices and it works well in Rapberry Pi. [OpenFaas](https://www.openfaas.com/) is a simple platform that can wrap functions written in many languages (Java, JavaScript, Python, Go, R, Shell script, Cobol, etc) into a Function as a Service modules and deploy to Kubernetes. The instructions for setting up k3s in Raspberry Pi cluster are taken from this article [Will it cluster? k3s on your Raspberry Pi](https://blog.alexellis.io/test-drive-k3s-on-raspberry-pi/)

#### Dashboards
Kubernetes is a command line driven system. The cloud system providers like Google Clouds, AWS provide useful dashboards to manage kubernetes resources and operations.

[Kubernetes Dashboard](https://github.com/kubernetes/dashboard) can be installed in the Raspberry Pi cluster to serve as UI.

Side note:
[OpenShift](https://www.okd.io/) is a 'Platform as a Service' offering that is built upon Kubernetes and has very powerful dashboard along with CLI.

#### Single Node setup
Single node ready to run VM instances are available for Kubernetes as [minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/) and for OpenShift as [minishift](https://github.com/minishift/minishift).  Which are useful to explore these platforms. 

### Quarkus
[Quarkus](https://quarkus.io/) is a comprehensive Java framework that compiles Java code into container applications suitable for 'Function as a Service' modules to start up quickly and use less resources.

#### Quick start
[Link](https://quarkus.io/guides/getting-started-guide)
**Prerequsites**
1. JDK 1.8+ installed with JAVA_HOME configured appropriately
2. Apache Maven 3.5.3+

**Bootstrapping the project**
```sh
mvn io.quarkus:quarkus-maven-plugin:0.18.0:create \
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

__skipping local testing__

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

The [run-java.sh](./docs/run-java.sh) is extracted from the fabric8/java-alpine-openjdk8-jre image. We need to copy this file to the project directory where pom.xml is found. 

After changing the Dockerfile and copying the run-java.sh, the build command generates the docker image that can successfully run in Raspberry Pi.


### References
- [Raspberry Pi Dramble](https://www.pidramble.com/)
- [CNCF Presentations](https://github.com/cncf/presentations/tree/master/kubernetes)
- [k3s Lightweight Kubernetes](https://k3s.io/)
- [Quarkus](https://quarkus.io/)
- [kubernetes Documentation](https://kubernetes.io/docs/home/)
- [OpenHab](https://www.openhab.org/)
- [Janakiram](https://www.youtube.com/user/janakirammsv)
