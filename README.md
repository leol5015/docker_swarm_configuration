#Docker Swarm with Multicast over Local Area Network
## Introduction
 This document was motivated by the desire to use a docker to deploy a cluster of micro
 services. Specifically we wanted to use the Vert.x stack. Unfortunately we ran into problems
 frequently along the path. To begin with almost all docker documentation is either writen for toy
 examples that either only use docker instance or all the instances are virtual machines run in
 docker toolbox on a local machine. Which is nice from an  ease of use perspective but not the most
 help full when trying to asses how to configure a custom cluster on dedicated hardware. Docker also
 has several documents that cover how to set up more production ready setups, but these examples are
 almost exclusively made for AWS or similar cloud services. Which introduces a lot of extra details
 that confuse the core information, for people that for one reason or another need to host their
 systems from a old fashioned datacenter.

 The next hurdle we ran into was dockers limitations when it comes to using multicast communication
 for service discovery which is the default behaviour for Vert.x. It is possible to configure
 discovery without multicast but this requires enumerating ip addresses that we should connect against.
 Which is a model that is not well suited to a docker swarm as we do not know the ip addresses of
 the individual containers until after they have been deployed. Instead we want to configure the
 docker swarm to emulate a local network so we can keep our applications as simple as possible.

 Hence the main goal of our setup is to construct a virtual network spanning multiple docker hosts
 that supports both point to point connections and multicast.  At the time of writing docker does not
 natively provide such a network. The default bridge network can handle multicast but not between
 multiple hosts. Docker also provides the overlay network type, this network can send point to point
 messages over ip between containers on different hosts, but does not support multicast.
 Luckily Docker supports network plugins. We will use one such plugin called weave net.

 The four components of our setup will thus be:
 * Docker to provide a way to standardise and isolate our applications.
   https://www.docker.com/
 * Docker swarm provides a central control node for managing application containers.
   https://docs.docker.com/swarm/
 * Consul provides a clustered key value store that allows the swarm to keep track of its members.
   https://www.consul.io/
 * Weave net provides a docker network plugin that allows our applications to communicate with each other.
   https://www.weave.works/


 We will not be using docker machine as we do not want to introduce any more virtualization that we
 have to. We also do not really bother to secure the communication over the cluster.
 The reason for this is that we want to keep this setup as simple as possible. And our intended
 use case is a secure local network such as a company owned server rack in a datacenter or a
 internal office network. If the setup should need to be used in a less secure network, both weave
 and swarm support tls encrypted communication but as that complicates the setup we will not use
 that in this initial guide.


## Consul
 The first component we will set up is consul. It is possible to run consul in a docker container
 but the setup is not overly complicated and doing it natively feels more correct.
 Installing the consul agent has to be done manually as it does not seem to be available in the
 standard apt or yum repositories but the installation is relatively simple.

 It is recommended that a consul cluster contains five servers for a production setup, the reason
 for this is that two machines can be down without compromising the clusters integrity.
 So if for instance we were updating one  machine when another failed the cluster would still be
 intact and recover automatically when we brought up the two machines that were down.
 For development a single machine or a three machine setup is perfectly fine as we do not care about
 uptime in the same way.

#### Installing

 In this example we assume that we will run a consul server on each docker host. Go to
 https://www.consul.io/downloads.html
 and copy the download link for the relevant platform.

 Then for each server that needs to run consul run the following.
```
 > curl -L <link_to_download> -o consul.zip
 > unzip consul.zip
 > sudo cp consul /usr/local/bin/consul
 > sudo chmod +x /usr/local/bin/consul
```

#### Configuring

 Next create a folder `/var/consul` that the agent will use to store persistent data. If we need to
 change this path the `data_dir` value in the following configs should be changed to mach the new folder.
 The chosen folder needs to be persistent across reboots and needs to support filesystem locking.


 On one machine in the cluster create `/etc/consul.d/bootstrap/config.json` with the following content:
```json
{
    "bootstrap": true,
    "server": true,
    "datacenter": "aha",
    "data_dir": "/var/consul",
    "encrypt": "PVcExGWGZeYR79CiRUT4vA==",
    "log_level": "INFO",
    "enable_syslog": true,
    "ui":true
}
```

 On each host that will run a consul server create a file `/etc/consul.d/server/config.json` with the
 following content except that we update the ip addresses so that `start_join` contains all the other server
 addresses and `client_addr` as well as `bind_addr` contains this servers ip:
```json
{
    "bootstrap": false,
    "server": true,
    "datacenter": "aha",
    "data_dir": "/var/consul",
    "encrypt": "PVcExGWGZeYR79CiRUT4vA==",
    "log_level": "INFO",
    "enable_syslog": true,
    "start_join": ["10.0.1.106", "10.0.1.111"],
    "client_addr":"10.0.1.104",
    "bind_addr":"10.0.1.104",
    "ui":true
}
```

 On each computer that will run client instances (normally not needed in a LAN setup) create a file
 `/etc/consul.d/client/config.json` with the following content replacing the ips in start_join
 with all the server ips, and `client_addr` as well as `bind_addr` with the computers ip:
```json
{
    "bootstrap": false,
    "server": false,
    "datacenter": "aha",
    "data_dir": "/var/consul",
    "encrypt": "PVcExGWGZeYR79CiRUT4vA==",
    "log_level": "INFO",
    "enable_syslog": true,
    "start_join": ["10.0.1.104", "10.0.1.106", "10.0.1.111"],
    "client_addr":"10.0.1.17",
    "bind_addr":"10.0.1.17",
    "ui":true
}
```

 Note the encrypt value in each config specifies the secret key that the cluster uses to encrypt
 network traffic. It must be 16-bytes that are Base64-encoded, the easiest way generate such a key
 is to call `consul keygen`. All nodes in a cluster must share the same encrypt key.

 The example configs for server exposes the web ui on port 8500 by setting client_addr to the server ip.
 If client_addr is omitted from the config the ui will be accessible from 127.0.0.1:8500 from local
 the local machine but not exposed on the network.

 For the meaning of each option or more advanced configuration check the consul agent options page
 in the documentation:
 https://www.consul.io/docs/agent/options.html

#### systemd Service

 Next we need to create a systemd service for consul.  On each server create a file
`/lib/systemd/system/consul.service` with the following content:
```
[Unit]
Description=Consul server service
Documentation=https://www.consul.io/docs/index.html
After=nerwork.target
Before=docker.service

[Service]
Type=simple
Environment=SYSTEMD_LOG_LEVEL=debug
ExecStart=/usr/local/bin/consul agent -ui -config-dir /etc/consul.d/server
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

 Finally enable consul in systemd so it starts on boot by running
```
> sudo systemctl enable consul.service
```

#### Bootstrapping
 Now we have set up all the configs that we need so all that remains is to bootstrap the cluster and
 set the servers to start up consul on boot. To do this log into the server where we added the
 `/etc/consul.d/bootstrap/config.json` file and run the following command.

```
> sudo consul agent -config-dir /etc/consul.d/bootstrap
```

 Then start the systemd service on the rest of the machines in the cluster by running
```
> sudo systemctl start consul.service
```

 Once the other servers have successfully started terminate the bootstrap process on the first server
 and instead start the consul.service just as we did on the rest of the servers.


## DOCKER
 Now we need to install docker engine on the servers. This process is straight forward and well
 documented on dockers web page.
 Simply follow the instructions for your distro on:
 https://docs.docker.com/engine/installation/

## WEAVE:
 Weave is a network plugin for docker that supports multicast. As such it provides the main
 capability that motivated this article. For further details see weaves website:
 https://www.weave.works/

### Installing Weave
 To install the latest version of weave simply run the following commands:
```
> sudo curl -L git.io/weave -o /usr/local/bin/weave
> sudo chmod a+x /usr/local/bin/weave
```

### Launch weave
 Now that we have the binary installed we can start up weave by running three simple commands,
 the first of which is:
```
> weave launch-router
```

 If the host is running a dns the router needs ot be launched with `weave launch-router --no-dns` to
 avoid conflict. The Weave binary will now download and launch the router as a docker container.

 Next we want to start up the weave proxy, the default is that the proxy will listen to either a
 unix socket or port 12375 depending on how the docker engine it is launched in is set up. But since we do
 not want to have to reconfigure the docker service and we need the port version to be able to connect
 to it from inside our swarm containers we will use the -H flag to force weave to expose the port.
 The command to start up the proxy then becomes:
 ```
 > weave launch-proxy -H tcp://<server_ip>:12375
 ```
 Where `<server_ip>` should be replaced with the desired ip to bind to on each server.

 After we have started the router and proxy on each server we want to connect them to each other.
 To do this se will use the `weave connect` command. The command takes a space separated list of all
 the ip addresses to connect to. So for instance if we had three servers with the ip  addresses
 10.0.1.1 10.0.1.2 and 10.0.1.3 we would run the following three commands to connect the network.

```
root@10.0.1.1> weave connect 10.0.1.2:12375 10.0.1.3:12375
```
```
root@10.0.1.2> weave connect 10.0.1.1:12375 10.0.1.3:12375
```
```
root@10.0.1.3> weave connect 10.0.1.1:12375 10.0.1.2:12375
```

 It is possible that weave could use a peer discovery system to find the other servers as we are
 running on a local network, but we define them anyway to make sure.

 Once you have connected the weave network we can check that it is configured correctly by running
```
> weave status
```

 Which should produce a result that looks roughly like this:
```

        Version: 1.5.0 (up to date; next check at 2016/04/26 13:23:31)

        Service: router
       Protocol: weave 1..2
           Name: ca:21:dc:bb:25:99(snapshot)
     Encryption: disabled
  PeerDiscovery: enabled
        Targets: 2
    Connections: 2 (2 established)
          Peers: 3 (with 6 established connections)
 TrustedSubnets: none

        Service: ipam
         Status: ready
          Range: 10.32.0.0-10.47.255.255
  DefaultSubnet: 10.32.0.0/12

        Service: dns
         Domain: weave.local.
       Upstream: 10.0.1.104
            TTL: 1
        Entries: 1

        Service: proxy
        Address: tcp://10.0.1.1:12375


```

 Make sure that peers is equal to the number of servers we connected and that proxy is the ip:port
 combination we provided to the launch-proxy command.

## Create docker swarm
 The final component that we need to set up are docker swarm containers. We need one swarm agent on
 each host and at least one swarm manager for the whole cluster. For convinience sake we will follow
 the same principle that we have so far and let each server have a manager. This allows us to manage
 the swarm from any host in the cluster which makes things easy but might not be what we want if we
 have a setting with stricter security procedures for instance.

 As we have set upp a consul and weave instance on each host the command to start a manager instance
 on a server with ip 10.0.1.1 will be the following.

 ```
 > docker run -d -p 4000:4000 --restart=always swarm manage -H :4000 --advertise 10.0.1.1:12375 consul://10.0.1.1:8500
 ```
 This will expose a new docker engine instance that listens on port 4000 and is connected to the
 swarm cluster.

 And to start the swarm nodes we would run the following command.
 ```
 > docker run -d --restart=always swarm join --advertise=10.0.1.1:12375 consul://10.0.1.1:8500
 ```

 Once we have run these two commands on each host in the cluster replacing 10.0.1.1 with the hosts
 ip we have completed our setup. To run a comand on the swarm instead of on the local docker engine
 simply prepend your docker command with '-H :4000' such that `docker ps -a` becomes
 `docker -H :4000 ps -a`

## Testing

 The first step for verifying that we have everything up and running is to run docker info on the
 swarm engine. This should look something like this.

 ```
 leol@snapshot:~$ docker -H :4000 info
 Containers: 39
  Running: 33
  Paused: 0
  Stopped: 6
 Images: 48
 Server Version: swarm/1.2.0
 Role: primary
 Strategy: spread
 Filters: health, port, dependency, affinity, constraint
 Nodes: 2
  demo: 10.0.1.106:12375
   └ Status: Healthy
   └ Containers: 28
   └ Reserved CPUs: 0 / 16
   └ Reserved Memory: 0 B / 32.98 GiB
   └ Labels: executiondriver=, kernelversion=4.2.0-35-generic, operatingsystem=Ubuntu 15.10, storagedriver=aufs
   └ Error: (none)
   └ UpdatedAt: 2016-04-28T12:10:58Z
   └ ServerVersion: 1.11.0
  snapshot: 10.0.1.111:12375
   └ Status: Healthy
   └ Containers: 11
   └ Reserved CPUs: 0 / 8
   └ Reserved Memory: 0 B / 32.99 GiB
   └ Labels: executiondriver=, kernelversion=4.2.0-34-generic, operatingsystem=Ubuntu 15.10, storagedriver=aufs
   └ Error: (none)
   └ UpdatedAt: 2016-04-28T12:10:32Z
   └ ServerVersion: 1.11.0
 Plugins:
  Volume:
  Network:
 Kernel Version: 4.2.0-34-generic
 Operating System: linux
 Architecture: amd64
 CPUs: 24
 Total Memory: 65.97 GiB
 Name: e8266ea2ca99
 Docker Root Dir:
 Debug mode (client): false
 Debug mode (server): false
 ```

 Next we will test that we can use the weave dns to discover containers and send pings between two
 containers on different hosts. To do this we will start two containers and tell swarm to host them
 on different hosts. To do this we run the following to commands in different shells.
 ```
 > docker -H :4000 run  -e 'affinity:container!=pingtest2' -ti --rm --name=pingtest1 gliderlabs/alpine sh -l
 ```

 ```
 > docker -H :4000 run  -e 'affinity:container!=pingtest1' -ti --rm --name=pingtest2 gliderlabs/alpine sh -l
 ```

 We can then ping the other container by using the weave dns. In the pintest1 container run:
 ```
 pingtest1:/# ping -c3 pingtest2.weave.local
 PING pingtest2.weave.local (10.40.0.0): 56 data bytes
 64 bytes from 10.40.0.0: seq=0 ttl=64 time=0.402 ms
 64 bytes from 10.40.0.0: seq=1 ttl=64 time=0.340 ms
 64 bytes from 10.40.0.0: seq=2 ttl=64 time=0.304 ms

 --- pingtest2.weave.local ping statistics ---
 3 packets transmitted, 3 packets received, 0% packet loss
 round-trip min/avg/max = 0.304/0.348/0.402 ms
 ```
 And in pingtest2 we should get something like:
 ```
 pingtest2:/# ping -c3 pingtest1.weave.local
 PING pingtest1.weave.local (10.36.0.0): 56 data bytes
 64 bytes from 10.36.0.0: seq=0 ttl=64 time=1.313 ms
 64 bytes from 10.36.0.0: seq=1 ttl=64 time=0.298 ms
 64 bytes from 10.36.0.0: seq=2 ttl=64 time=0.359 ms

 --- pingtest1.weave.local ping statistics ---
 3 packets transmitted, 3 packets received, 0% packet loss
 round-trip min/avg/max = 0.298/0.656/1.313 ms
 ```

 The image also comes with nc installed which can be used for more advanced newtwork tests.

 Next we will test that a simple Vert.x app taken from their example repository can be run in
 containers, docsover each other with the default multicast mode and send messages over the
 eventbus. The example we will be running can be seen here:
 https://github.com/vert-x3/vertx-examples/blob/master/core-examples/README.adoc#publish--subscribe

 We have simply removed all references to io.vertx.example.util.Runner, striped the package
 declaration from the two source files and bundled them into a docker image.

 Which lets us run the following two commands from two shells:

 ```
 > docker -H :4000 run -ti --rm --name=bus_receiver leol/vertx_eventbus vertx run Receiver.java -cluster
 ```
 ```
 > docker -H :4000 run -e 'affinity:container!=bus_receiver' -ti --rm --name=bus_sender leol/vertx_eventbus vertx run Sender.java -cluster
 ```

 Which should give us output in the following form:
 ```
 Starting clustering...
 No cluster-host specified so using address 10.40.0.0
 Ready!
 Succeeded in deploying verticle
 ```
 And once both containers have started a sstream of `Received news: Some news!` should start
 scrolling down the Receiver instance.





