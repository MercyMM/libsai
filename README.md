# Libsai
Libsai is a small SAI implementation for testing SAI applications.

While SAI provides general APIs for switch applications and vendors can have their own SDKs running with SAI and their switches, there is no way to
 - Easily test SAI applications
 - Without using real SAI-capable switches

Our goal is to mitigate these issues by providing open sourced SAI implementation that leverage software switches.

## How to build
Please refer to [doc/Building.md](doc/Building.md)

## How to run
Before staring, make sure you finished everything in the "How to build" section.

1. Run the switch VM

    ```
    host-OS$ sudo qemu-system-x86_64 [image of switch VM] -enable-kvm -device rocker,name=sw1,len-ports=4,ports[0]=dev0,ports[1]=dev1,ports[2]=dev2 -netdev bridge,br=br0,id=dev0 -netdev bridge,br=br1,id=dev1 -netdev bridge,br=br2,id=dev2
    ```

2. Run the three other VMs

    ````
    host-OS$ sudo qemu-system-x86_64 [image of VM1] -enable-kvm -net nic,macaddr=52:54:00:00:00:11 -net bridge,br=br0
    host-OS$ sudo qemu-system-x86_64 [image of VM2] -enable-kvm -net nic,macaddr=52:54:00:00:00:12 -net bridge,br=br1
    host-OS$ sudo qemu-system-x86_64 [image of VM3] -enable-kvm -net nic,macaddr=52:54:00:00:00:13 -net bridge,br=br2
    ````
The three VMs are attached to the switch VM as described in figure.
Note that the VMs are still not yet "switched", thus no pakcets can be exchanged even if you try to set addresses to the VMs.
![three VMs are attached to the switch VM](doc/libsai_VM_attached.png)

3. Assing addresses to the VMs, and execute libsai in the switch VM.

    ````
    VM1$ sudo ip addr add 192.168.1.2/24 dev eth0
    VM1$ sudo ip link set up eth0
    VM1$ sudo ip route add 192.168.2.0/24 via 192.168.1.1 dev eth0

    VM2$ sudo ip addr add 192.168.1.3/24 dev eth0
    VM2$ sudo ip link set up eth0
    VM2$ sudo ip route add 192.168.2.0/24 via 192.168.1.1 dev eth0

    VM3$ sudo ip addr add 192.168.2.2/24 dev eth0   # be careful it's 2.2, not 1.2 nor 1.4
    VM3$ sudo ip link set up eth0
    VM3$ sudo ip route add 192.168.1.0/24 via 192.168.2.1 dev eth0  # be careful again, VM3 is under 192.168.2.0/24
    
    switch-VM $ cd libsai
    switch-VM $ sudo ./a.out
    switch-VM $ sudo su -
    switch-VM $ echo "1" > /proc/sys/net/ipv4/ip_forward
    ````
At this point, the three VMs are connected and routed under the topology in the figure below.
VM1 and VM2 are connected to the same VLAN (saivlan), and VM3 is connected to another VLAN.
The two VLANs are connected to virtual router (sairouter) and routed each other so that VMs in different VLANs can communicate each other.
![three VMs are connected and routed under the topology](doc/libsai_VM_connected.png)

4. Try ping between the VMs and see what is happening.
 - **Ping from VM1 to VM2 while executing `tcpdump -vvv` on VM2**: You will first see an ARP request arriving to VM2 from VM1, and then ICMP packets start arriving. This is because VM1 and VM2 are connected to the same VLAN. Note that you cannot see the VLAN tags on the VMs because they are automatically pushed/dropped inside the switch VM.
 - **Ping from VM1 to VM3 while executing `tcpdump -vvv` on VM3**: You will again see an ARP request arriving to VM3, but this time not from VM1 but from 192.168.2.1. This is because VM1 and VM3 are connected to different VLANs and the two VLANs are routed on L3.

## How to hack
In main.c the topology used above is intutively coded.

    // initialize the switch
    sw = switch_init();

    ......
    
    Vlan* vlan1 = create_vlan(2, &sw->ports[0], &sw->ports[1]);
    Vlan* vlan2 = create_vlan(2, &sw->ports[2], &sw->ports[3]);

    // create a virtual router
    Router* router = create_router(sw);

    // assign 192.168.1.0/24 to vlan1
    add_vlan_to_router(router, vlan1, make_ip4("192.168.1.1"));
    create_new_route(router, make_ip4("192.168.1.0"), make_ip4("255.255.255.0"), vlan1);

    // assign 192.168.2.0/24 to vlan2
    add_vlan_to_router(router, vlan2, make_ip4("192.168.2.1"));
    create_new_route(router, make_ip4("192.168.2.0"), make_ip4("255.255.255.0"), vlan2);

You can create as many VLANs as you want and create more complex routes by using these abstractions.
Note that as this project is still in its preliminary stage, no other SAI APIs (acl, samplepacket, QoS, ...) are not yet supported.

If you want to do something by directily touching SAI APIs, see libsai.c and libsai.h

If you want to know what it does between SAI APIs and rocker device, see rocker/*.c

## Copyright
Copyright (C) 2015 [Nippon Telegraph and Telephone Corporation](http://www.ntt.co.jp/index_e.html). Released under [Apache License 2.0](LICENSE).
