# Extend SRIOV VFs to Containers

## Problem Statment
[SRIOV_Basics](https://www.intel.com/content/www/us/en/developer/articles/technical/configure-sr-iov-network-virtual-functions-in-linux-kvm.html)
* While supporting design/ deployment and operations of NFVI solutions with multiple customers I had extensive hands on experience with SRIOV.
* The need was obvious, that Communication service provider always have work loads which require real time throughput, thus demand to bypass the (Qemu emulation of network devices) and have direct networking capabilities inside VNFs via SRIOV VFs. 
* During recent learning endeavours to prepare a NFVI solution based on K8s cluster running on bare metals  I  faced a challenge on how to extending SRIOV VFs to K8s pods / containers as it was relatively easy to extend SRIOV VFs to Openstack VMs.
## Solution
* SRIOV CNI  solves the above described problem, but there were  many questions e.g. 
  - How a SRIOV VF can be extended to a Container / POD.
  - How  to do traffic segregation via  separate VLANs  and on  each SRIOV VF.
* As high rises are always made of small and basic building blocks so I focused on above 2 questions and found their answers which is further described in this wiki. 
* At high level solution would be:-
  - Enable the SRIOV functionality on bare metal server. 
  - Create required number of virtual functions (VFs) over SRIOV supported NIC.
  - Move the SRIOV VF to running container's namespace and attach it with the running container.
![SRIOV_VFs_inside_Containers](./images/SRIOV_VFs_inside_Containers.jpg)

## Implementation Details 

![Lab_Diagram](./images/Lab_Diagram.jpg)

* Check your NIC hardware 

  ```
  sudo lshw -c network -businfo 

  sudo lshw -c network -businfo
  Bus info          Device       Class          Description
  =========================================================
  pci@0000:01:00.0  eno1         network        Ethernet Controller 10-Gigabit X540-AT2
  pci@0000:01:00.1  eno2         network        Ethernet Controller 10-Gigabit X540-AT2
  pci@0000:07:00.0  eno3         network        I350 Gigabit Network Connection
  pci@0000:07:00.1  eno4         network        I350 Gigabit Network Connection
  ```
* Enable SRIOV functionality 
  ```
  sed -i s/GRUB_CMDLINE_LINUX_DEFAULT=""/GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt"/ /etc/default/grub
  update-grub
  reboot
  ```
* Create SRIOV VFs

  ```
  cat << EOF > /etc/rc.local
  #!/bin/bash
  #echo 8 > /sys/class/net/eno1/device/sriov_numvfs
  echo 8 > /sys/class/net/eno2/device/sriov_numvfs
  EOF

  cat << EOF > /etc/systemd/system/rc-local.service
  [Unit]
  Description=/etc/rc.local Compatibility
  ConditionPathExists=/etc/rc.local

  [Service]
  Type=forking
  ExecStart=/etc/rc.local start
  TimeoutSec=0
  StandardOutput=tty
  RemainAfterExit=yes
  SysVStartPriority=99

  [Install]
  WantedBy=multi-user.target
  EOF

  sudo chmod +x /etc/rc.local
  sudo systemctl start rc-local.service
  sudo systemctl status rc-local.service
  sudo systemctl enable rc-local
  ```
* Verify SRIOV VFs

  ```
  sudo lshw -c network -businfo
  Bus info          Device       Class          Description
  =========================================================
  pci@0000:01:00.0  eno1         network        Ethernet Controller 10-Gigabit X540-AT2
  pci@0000:01:00.1  eno2         network        Ethernet Controller 10-Gigabit X540-AT2
  pci@0000:01:10.1  eno2v0       network        X540 Ethernet Controller Virtual Function
  pci@0000:01:10.3  eno2v1       network        X540 Ethernet Controller Virtual Function
  pci@0000:01:10.5  eno2v2       network        X540 Ethernet Controller Virtual Function
  pci@0000:01:10.7  eno2v3       network        X540 Ethernet Controller Virtual Function
  pci@0000:01:11.1  eno2v4       network        X540 Ethernet Controller Virtual Function
  pci@0000:01:11.3  eno2v5       network        X540 Ethernet Controller Virtual Function
  pci@0000:01:11.5  eno2v6       network        X540 Ethernet Controller Virtual Function
  pci@0000:01:11.7  eno2v7       network        X540 Ethernet Controller Virtual Function
  pci@0000:07:00.0  eno3         network        I350 Gigabit Network Connection
  pci@0000:07:00.1  eno4         network        I350 Gigabit Network Connection
  ```
* SRIOV VFs are created over eno2 NIC, v0 to v7 virtual functions can be seen in the above snippet.

* Enable VLAN Tagging Over SIROV VFs

  ```
  ip link set dev eno2 vf 0 vlan 200
  ip link set dev eno2 vf 1 vlan 201
  ```

* Create the Conainers

  ```
  docker run -d --name test-200 nginx:alpine
  docker run -d --name test-202 nginx:alpine
  ```
* Above snippet will create 2 containers with default network driver, i.e. bridge and will attach 1 interface to each container (besides lo0).

* Attach the SRIOV VFs to Container namespaces.  

  ```
  namespace=$(docker inspect test-200 -f '{{.State.Pid}}')
  mkdir -p /var/run/netns/
  ln -sfT /proc/$namespace/ns/net /var/run/netns/$namespace
  HOST_IF=eno2v0
  C_IF=eth1 
  ip link set $host_if netns  $namespace
  ip netns exec $namespace ip link set $HOST_IF name $C_IF
  ip netns exec $namespace ip link set $C_IF up
  ip netns exec $namespace ip addr add 192.168.200.2/24 dev $C_IF


  HOST_IF=eno2v1
  C_IF=eth1 
  namespace=$(docker inspect test-201 -f '{{.State.Pid}}')
  ln -sfT /proc/$namespace/ns/net /var/run/netns/$namespace
  ip netns exec $namespace ip link list
  ip link set $HOST_IF netns  $namespace
  ip netns exec $namespace ip link set $HOST_IF name $C_IF
  ip netns exec $namespace ip addr add 192.168.201.2/24 dev $C_IF
  ip netns exec $namespace ip link set $C_IF up
  ```
* Verifcations 

  ```
  docker exec -it test-200 /bin/sh
  / # ip a
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
  6: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
      link/ether 9a:88:8c:f7:bf:de brd ff:ff:ff:ff:ff:ff
      inet 192.168.200.2/24 scope global eth1
        valid_lft forever preferred_lft forever
  15: eth0@if16: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
      link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
      inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
        valid_lft forever preferred_lft forever

  / # ping -c1 192.168.200.1
  PING 192.168.200.1 (192.168.200.1): 56 data bytes
  64 bytes from 192.168.200.1: seq=0 ttl=64 time=2.003 ms


  docker exec -it test-201 /bin/sh

  lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
  7: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
      link/ether de:bb:04:86:b8:36 brd ff:ff:ff:ff:ff:ff
      inet 192.168.201.2/24 scope global eth1
        valid_lft forever preferred_lft forever
  19: eth0@if20: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
      link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
      inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
        valid_lft forever preferred_lft forever
  / # ping -c1 192.168.201.1
  PING 192.168.201.1 (192.168.201.1): 56 data bytes
  64 bytes from 192.168.201.1: seq=0 ttl=64 time=3.310 ms


  docker exec -it test-201 arp -a
  ? (192.168.201.1) at ac:4b:c8:2b:77:c1 [ether]  on eth1
  ? (192.168.201.1) at ac:4b:c8:2b:77:c1 [ether]  on eth1
  root@worker2:~# docker exec -it test-200 arp -a
  ? (192.168.200.1) at ac:4b:c8:2b:77:c1 [ether]  on eth1
  ? (192.168.200.1) at ac:4b:c8:2b:77:c1 [ether]  on eth1

  show ethernet-switching table interface ge-0/0/1
  Ethernet-switching table: 2 unicast entries
    VLAN	            MAC address       Type         Age Interfaces
    SRIOV_200         *                 Flood          - All-members
    SRIOV_200         9a:88:8c:f7:bf:de Learn       4:05 ge-0/0/1.0
    SRIOV_201         *                 Flood          - All-members
    SRIOV_201         de:bb:04:86:b8:36 Learn          0 ge-0/0/1.0

  show ethernet-switching table interface ge-0/0/1
  Ethernet-switching table: 2 unicast entries
    VLAN	            MAC address       Type         Age Interfaces
    SRIOV_200         *                 Flood          - All-members
    SRIOV_200         9a:88:8c:f7:bf:de Learn       4:05 ge-0/0/1.0
    SRIOV_201         *                 Flood          - All-members
    SRIOV_201         de:bb:04:86:b8:36 Learn          0 ge-0/0/1.0


  show arp no-resolve | grep 192.168.200
  9a:88:8c:f7:bf:de 192.168.200.2   vlan.200             none 
  show arp no-resolve | grep 192.168.201
  de:bb:04:86:b8:36 192.168.201.2   vlan.201             none
  ```


