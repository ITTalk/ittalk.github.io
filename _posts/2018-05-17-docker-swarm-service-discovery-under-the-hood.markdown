---
layout: default
title:  "Docker Swarm service discovery under the hood"
date:   2018-05-17
categories: docker swarm service-discovery
post_author: Daniel Musiał
---

Recently I've been facing a couple of issues with Docker Swarm built-in DNS and service discovery mechanisms which made me look under the hood to see what's going on. This post is meant to quickly summarize my findings for future use and also help you understand better how Docker's service discovery is built internally. If you're looking for a more introductory course on the topic, there's plenty resources available freely online.

## Lab preparation

To help you get up and running quickly, I've prepared a Vagrantfile and a couple of shell scripts that will get you from nothing to a fully configured 2-node docker swarm in no time. The only thing you need is Vagrant + VirtualBox installed on your machine. You can find all the files on [Github](https://github.com/dmusial/DockerSwarmLab). Once you clone the repo, just run `vagrant up` inside the main folder and give it a couple of minutes to complete. When completed, you can SSH into the nodes by running `vagrant ssh docker-manager01` or `vagrant ssh docker-worker01`.

Alternatively you can try using online services like [Play-With-Docker](https://labs.play-with-docker.com/), that make testing things even easier.

## Testing service discovery

Now that we have a running lab environment, we'll need a few sample services to test discovery. First, let's create an overlay network scoped to swarm that we'll use to attach our services (containers) to:

```
$ sudo docker create network --driver overlay my-ov
$ sudo docker network ls
```
Next, let's create two services and attach them to our newly created overlay network:
```
$ sudo docker service create --name svc1 --network my-ov alpine sleep 1d
$ sudo docker service create --name svc2 --network my-ov alpine sleep 1d
$ sudo docker service ls
```

For the sake of simplicity, I'm creating small alpine-based containers that I put immediatelly into sleep for one day just to keep them alive. To confirm that the service discovery is indeed working, let's try resolving service's IP address from within one of the containers using `nslookup` via `docker exec`:
```
$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
df6ab5015deb        alpine:latest       "sleep 1d"          13 minutes ago      Up 42 minutes                           svc2.1.dsbm8gqzkb9155w98cdkm72lo
3b78d0bc2206        alpine:latest       "sleep 1d"          2 minutes ago       Up 42 minutes                           svc1.1.698qak908bz8g9ab39xptn7ho

$ sudo docker exec df6ab5015deb nslookup svc1
Name:      svc1
Address 1: 10.0.0.15
```

The resolved IP 10.0.0.15 is in fact service's VIP. We can confirm that by running:
```
$ sudo docker service inspect -f '{{.Endpoint.VirtualIPs}}' svc1
[{1noean4e8708otnvxz6wxpy4f 10.0.0.15/24}]
```

Let's try testing name resolution for an individual container:
```
$ sudo docker exec df6ab5015deb nslookup svc1.1.698qak908bz8g9ab39xptn7ho
Name:      svc1.1.698qak908bz8g9ab39xptn7ho
Address 1: 10.0.0.16 svc1.1.698qak908bz8g9ab39xptn7ho.my-ov
```
Again, without any issues we get a correctly resolved container IP address. To confirm it is actually a correct IP lets run:
```
$ sudo docker inspect -f '{{ $network := index .NetworkSettings.Networks "my-ov" }}{{$network.IPAddress}}' 3b78d0bc2206
10.0.0.16
```

## How does service discovery work in Docker Swarm?

There's not much in-depth information on service discovery inside Docker documentation. We can learn however that every container attached to a user-defined network will get an extra service, DNS resolver, that will listen on 127.0.0.11 for incomming DNS queries [Source](https://docs.docker.com/v17.09/engine/userguide/networking/configure-dns/). Let's validate that on our lab environment.

I'm using an out-of-the-box alpine-based container from the previous example. Firstly, let's check container's DNS config:

```
$ sudo docker exec df6ab5015deb cat /etc/resolv.conf
nameserver 127.0.0.11
options single-request-reopen ndots:0
```

All DNS requests will indeed go to the address mentioned in the documentation. Is there anything listening on that address?

```
$ sudo docker exec df6ab5015deb netstat -lntu
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 127.0.0.11:42439        0.0.0.0:*               LISTEN
udp        0      0 127.0.0.11:48233        0.0.0.0:*
```

Everything as expected so far. The only question left is, how are regular DNS queris ending up in something that listens on port 48233 instead of the standard 53? Also, if we take a quick look at container's interfaces, we'll notice that there's actually no interface configured with IP 127.0.0.11:

```
$ sudo docker exec df6ab5015deb ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet 10.0.0.8/32 brd 10.0.0.8 scope global lo
       valid_lft forever preferred_lft forever
13: eth0@if14: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue state UP
    link/ether 02:42:0a:00:00:04 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.4/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
15: eth1@if16: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.3/16 brd 172.18.255.255 scope global eth1
       valid_lft forever preferred_lft forever
```

This means that something is probably performing network address translation. The most obvious candidate that first comes to my mind is `iptables`. I'm going to use `nsenter` to run `iptables` in the context of the namespace created for one fo my containers:

```
$ sudo docker inspect -f '{{.State.Pid}}' df6ab5015deb
12434
$ sudo nsenter -t 12434 -n iptables -t nat -L -n | grep 53
DNAT       tcp  --  0.0.0.0/0            127.0.0.11           tcp dpt:53 to:127.0.0.11:42439
DNAT       udp  --  0.0.0.0/0            127.0.0.11           udp dpt:53 to:127.0.0.11:48233
SNAT       tcp  --  127.0.0.11           0.0.0.0/0            tcp spt:42439 to::53
SNAT       udp  --  127.0.0.11           0.0.0.0/0            udp spt:48233 to::53
```

Docker has automatically created DNAT and SNAT rules that help DNS queries to flow to the resolver inside the container. Even though, from the outside, it might seem like the resolver has all the knowledge to resolve service names, all it really does is forward requests to docker daemon running on the host. Docker daemon uses an internal distributed key-value store to retrieve service information and returns it back to the resolver inside the container. If the daemon fails to find required records in the KV store, it forwards further to external DNS server.


![Docker Swarm Service Discovery](/assets/images/aaa/DNS.png)

Source: [Docker](https://success.docker.com/article/ucp-service-discovery)

KV store plays a key part in Docker's service discovery mechanism. I'll dedicate a separate post in the near future to tell a bit about how to interact with it.

More reading:
 * [Docker Reference Architecture: Universal Control Plane 2.0 Service Discovery and Load Balancing](https://success.docker.com/article/ucp-service-discovery)