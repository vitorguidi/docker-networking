### IP command

* Responsible for managing interfaces, tunnels, network namespaces, network devices and route tables
* ip a => shows all devices
* ip ns => shows all network namespaces
* ip route => shows route tables 

### Docker

* Docker networking => network namespace
* Three main networking modes: host network, bridge (internal network) and overlay (idk wtf this is)

## Namespaces

* Uses network namespaces and veth interfaces
* Veth interfaces come in pairs. One end within netns, another within the host and traffic goes both ways
* Bridge network will unite all veth host connections and thus all docker containers will be connected

ip netns add $ns1 => creates net ns 
ip netns list
ip netns delete

* ip command allows us to execute commands within an inner namespace
* ip netns exec $ns1 ip (insert command here)

Gameplan:
* Create veth pair inside default namespace
* Move one of the interfaces to the namespace in question
* Add ip addresses to both interfaces
* Check connectivity

ip netns add ns1
ip link add veth-lol1 type veth peer name veth-lol2

Result: 

'''
22: veth-lol2@veth-lol1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether a6:c9:56:b8:f6:56 brd ff:ff:ff:ff:ff:ff
23: veth-lol1@veth-lol2: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 06:b3:f9:9a:db:14 brd ff:ff:ff:ff:ff:ff
'''

ip link set veth-lol2 netns ns1
ip netns exec ns1 ip link

'''
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
22: veth-lol2@if23: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether a6:c9:56:b8:f6:56 brd ff:ff:ff:ff:ff:ff link-netnsid 0
'''

cool, it moved

sudo ip addr add 192.168.49.1/24 dev veth-lol1
sudo ip netns exec ns1 ip addr add 192.168.49.2/24 dev veth-lol2

ip -n ns1 route add 192.168.50.0/24 via 192.168.49.2
ip -n ns2 route add 192.168.49.0/24 via 192.168.50.2


ping 192.168.49.2

voila

## Bridge


ip netns add ns2
ip link add veth-lol3 type veth peer name veth-lol4
ip link set veth-lol4 netns ns2

(Caveat: place 2nd namespace in another subnet else will have conflicting entries at route table)

sudo ip addr add 192.168.50.1/24 dev veth-lol3
sudo ip -n ns2 addr add 192.168.50.2/24 dev veth-lol4

ip link dev veth-lol3 set up
ip -n ns2 link dev veth-lol4 set up

ip link add name mockbr type bridge
ip link set dev veth-lol1 master mockbr
ip link set dev veth-lol3 master mockbr

ip netns exec ns1 ping 192.168.50.2
ip netns exec ns2 ping 192.168.49.2
profit


Source: https://docker-k8s-lab.readthedocs.io/en/latest/docker
https://www.youtube.com/watch?v=oVu0O0UMBCc&list=PLmZU6NElARbZtvrVbfz9rVpWRt5HyCeO7&ab_channel=RouterologyBlog