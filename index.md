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
These are the notes taken when setting up Rapberry Pi cluster for Kubernetes and running Quarkus. Overall the hardware setup is based on [Raspberry Pi Dramble](https://www.pidramble.com/) and Kubernetes setup is done based on [OpenFaas](https://blog.alexellis.io/test-drive-k3s-on-raspberry-pi/). Please follow them for detailed instructions.

Typically the local clusters are demonstrated using a dedicated router. Here we are using shared network setup on Ubuntu host, wired to the Ethernet switch of the cluster.

### Hardware
 - Rabpberry Pi - 4
 - Micro SD card - 4 
 - 6 inch Ethernet cable - 4
 
### Power Supply
- USB Power Source
  - USB Power Hub - 1
  - USB to Micro USB cable - 4
  - 4 port Ethernet Switch - 1
0r
- Power Over Ethernet
  - 4 port POE Ethernet Switch - 1

### Software

#### Operating System
- Standard raspbian (32 bit)
- Etcher


Quarkus support native compilation using GraalVM. However GraalVM is available for ARM processers that are used in Raspberry yet. One may try installing 64 bit Linux for ARM in the Raspberry Pi and compile GraalVM from source.


#### Networking
We have used the following IP address for the nodes.

  | host name | IP address |
  |-----------|------------|
  | master0   | 10.0.1.60  |
  | worker1   | 10.0.1.61  |
  | worker2   | 10.0.1.62  |
  | worker3   | 10.0.1.63  |

It is easy to setup shared Network connection to wired Ethernet in Ubuntu [Instructions] (https://askubuntu.com/questions/3063/share-wireless-connection-with-wired-ethernet-port).
However the address range used by default (10.42.0.x) did not match the static IP addresses selected for the nodes.
To change the IP range do the following. Further details are [here](https://askubuntu.com/questions/1062617/cannot-change-address-range-10-42-0-x-in-shared-to-other-computer-method).

```sh
sudo nmcli c modify 'Shared' ipv4.address 10.0.1.1/24
```

#### Kubernetes
We are using [k3s](https://k3s.io/) a lightweight distribution of Kubernetes targeted for edge devices and it works well in Rapberry Pi. [OpenFaas](https://www.openfaas.com/) is a simple platform that can wrap functions written in many languages (Java, JavaScript, Python, Go, R, Shell script, Cobol, etc) into a Function as a Service modules and deploy to Kubernetes. The instructions for setting up k3s in Raspberry Pi cluster are taken from this article [Will it cluster? k3s on your Raspberry Pi](https://blog.alexellis.io/test-drive-k3s-on-raspberry-pi/)

#### Quarkus
[Quarkus](https://quarkus.io/) is a comprehensive Java framework that compiles Java code into container applications suitable for Function as a Service modules to start up quickly and use less resources.



### References
- [Raspberry Pi Dramble](https://www.pidramble.com/)
- [CNCF Presentations](https://github.com/cncf/presentations/tree/master/kubernetes)
- [k3s Lightweight Kubernetes](https://k3s.io/)
- [Quarkus](https://quarkus.io/)
- [kubernetes Documentation](https://kubernetes.io/docs/home/)
- [OpenHab](https://www.openhab.org/)
- [Janakiram](https://www.youtube.com/user/janakirammsv)
