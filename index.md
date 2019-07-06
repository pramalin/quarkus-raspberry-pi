## Quarkus on Raspberry PI nodes
These are instructions for setting up Quarkus in Rapberry Pi cluster.

* [Quarkus on Raspberry PI nodes](#quarkus-on-raspberry-pi-nodes)
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
- raspbian (32 bit)
- Etcher

May try
- 64 bit Linux for ARM


#### Networking
Static IP address
   master0 - 10.0.1.60
   worker1 - 10.0.1.61
   worker2 - 10.0.1.62
   worker3 - 10.0.1.63

Host:
Ubuntu -
Shared Network
 Change IP range
Change Shared internet connection in Ubuntu
https://askubuntu.com/questions/1062617/cannot-change-address-range-10-42-0-x-in-shared-to-other-computer-method

```sh
sudo nmcli c modify 'Shared' ipv4.address 10.0.1.1/24
```

#### Kubernetes


#### Quarkus


#### Examples


### Other Cluster setups
- Drupal
- Casandra
- Spark
- OpenFaas

### 
- OpenHab


### References
- [Raspberry Pi Dramble](https://www.pidramble.com/)
- [CNCF Presentations](https://github.com/cncf/presentations/tree/master/kubernetes)
- [k3s Lightweight Kubernetes](https://k3s.io/)
- [Quarkus](https://quarkus.io/)
- [kubernetes Documentation](https://kubernetes.io/docs/home/)
- [OpenHab](https://www.openhab.org/)
- [Janakiram](https://www.youtube.com/user/janakirammsv)
