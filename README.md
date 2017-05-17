Start up the 3 vagrant managed CORE-OS instances (which have docker pre-installed)
=
```
$ vagrant up
```

NOTE: There is a supported CoreOS Vagrantfile here: https://github.com/coreos/coreos-vagrant/blob/master/Vagrantfile

SSH into first host and get IP address
=
```
$ vagrant ssh host-1
core@host-1 ~ $ ifconfig eth1 | grep 'inet ' | awk '{ print $2 }'
172.28.128.3
```

Each VM has other network adapters, but for now we’ll focus on this particular one. We can see that all 3 machines are part of the 172.28.128.0/24 network. On a production setup the different machines are probably not going to be on the same private network but we can still achieve this using virtual networks most of the time (VPC on AWS for instance).

Starting the Consul cluster
=

Start Consul on the three VM's in a cluster
==

Host-1
===
```
$(docker run --rm progrium/consul cmd:run 172.28.128.3 -d -v /mnt:/data)
```
Which is the same as running the following....but simpler
```
docker run -d -h node1 -v /mnt:/data \
-p 172.28.128.3:8300:8300 \
-p 172.28.128.3:8301:8301 \
-p 172.28.128.3:8301:8301/udp \
-p 172.28.128.3:8302:8302 \
-p 172.28.128.3:8302:8302/udp \
-p 172.28.128.3:8400:8400 \
-p 172.28.128.3:8500:8500 \
-p 172.17.42.1:53:53/udp \
progrium/consul -server -advertise 172.28.128.3 -bootstrap-expect 3
```

Host-2
===
```
$(docker run --rm progrium/consul cmd:run 172.28.128.4:172.28.128.3 -d -v /mnt:/data) 
```
Which is the same as running the following.....but simpler
```
docker run -d -h node2 -v /mnt:/data \
-p 172.28.128.4:8300:8300 \
-p 172.28.128.4:8301:8301 \
-p 172.28.128.4:8301:8301/udp \
-p 172.28.128.4:8302:8302 \
-p 172.28.128.4:8302:8302/udp \
-p 172.28.128.4:8400:8400 \
-p 172.28.128.4:8500:8500 \
-p 172.17.42.1:53:53/udp \
progrium/consul -server -advertise 172.28.128.4 -join 172.28.128.3
```


Host-3
===
```
$(docker run --rm progrium/consul cmd:run 172.28.128.5:172.28.128.3 -d -v /mnt:/data) 
```
Which is the same as running the following....but simpler
```
docker run -d -h node3 -v /mnt:/data \
-p 172.28.128.5:8300:8300 \
-p 172.28.128.5:8301:8301 \
-p 172.28.128.5:8301:8301/udp \
-p 172.28.128.5:8302:8302 \
-p 172.28.128.5:8302:8302/udp \
-p 172.28.128.5:8400:8400 \
-p 172.28.128.5:8500:8500 \
-p 172.17.42.1:53:53/udp \
progrium/consul -server -advertise 172.28.128.5 -join 172.28.128.3
```

Check that the Consul cluster is good
=
```
core@host-1 docker logs consul
```


Start the Registrator Docker Container
=
https://github.com/gliderlabs/registrator


Pull latest registrator
==
docker pull gliderlabs/registrator:latest

Start Registrator on Host-1
==
```
core@host-1 ~ $ export HOST_IP=$(ifconfig eth1 | grep 'inet ' | awk '{ print $2 }')
core@host-1 ~ $ docker run -d  \
                  --name=registrator \    
                   --net=host \   
                   --volume=/var/run/docker.sock:/tmp/docker.sock \
                   gliderlabs/registrator:latest \
                   consul://$HOST_IP:8500
```

Check that registrator is connected upto consul
==
```
$ docker logs registrator
2017/05/16 12:57:47 Starting registrator v7 ...
2017/05/16 12:57:47 Using consul adapter: consul://172.28.128.3:8500
2017/05/16 12:57:47 Connecting to backend (0/0)
2017/05/16 12:57:47 consul: current leader  172.28.128.3:8300
2017/05/16 12:57:47 Listening for Docker events ...
2017/05/16 12:57:47 Syncing services on 2 containers
2017/05/16 12:57:47 ignored: 225687414f81 no published ports
2017/05/16 12:57:47 added: 36df5ca0b79b host-1:consul:8500
2017/05/16 12:57:47 added: 36df5ca0b79b host-1:consul:8301:udp
2017/05/16 12:57:47 added: 36df5ca0b79b host-1:consul:8302:udp
2017/05/16 12:57:47 added: 36df5ca0b79b host-1:consul:53:udp
2017/05/16 12:57:47 added: 36df5ca0b79b host-1:consul:8300
2017/05/16 12:57:47 added: 36df5ca0b79b host-1:consul:8302
2017/05/16 12:57:47 added: 36df5ca0b79b host-1:consul:8400
2017/05/16 12:57:47 added: 36df5ca0b79b host-1:consul:53
2017/05/16 12:57:47 added: 36df5ca0b79b host-1:consul:8301
```

Start on the other two hosts.
==
```
core@host-2 ~ $ export HOST_IP=$(ifconfig eth1 | grep 'inet ' | awk '{ print $2 }')]
core@host-2 ~ $ docker run -d     --name=registrator     --net=host     --volume=/var/run/docker.sock:/tmp/docker.sock     gliderlabs/registrator:latest       consul://$HOST_IP:8500
```

```
core@host-3 ~ $ export HOST_IP=$(ifconfig eth1 | grep 'inet ' | awk '{ print $2 }')]
core@host-3 ~ $ docker run -d     --name=registrator     --net=host     --volume=/var/run/docker.sock:/tmp/docker.sock     gliderlabs/registrator:latest       consul://$HOST_IP:8500
```

Start CES Literal Filter Mock
=

https://github.com/jazzyray/cesliteralresponsefilter

Install Docker compose on the three hosts
==
```
core@host-1 ~ $ curl -L https://github.com/docker/compose/releases/download/1.13.0/docker-compose-`uname -s`-`uname -m` > ~/docker-compose
core@host-1 ~ $ sudo mkdir -p /opt/bin
core@host-1 ~ $ sudo mv ~/docker-compose /opt/bin/docker-compose
core@host-1 ~ $ sudo chown root:root /opt/bin/docker-compose
core@host-1 ~ $ sudo chmod +x /opt/bin/docker-compose
core@host-1 ~ $ whereis docker-compose
docker-compose: /opt/bin/docker-compose
```

GET the docker-compose.yml
```
core@host-1 ~ $wget https://raw.githubusercontent.com/jazzyray/cesliteralresponsefilter/master/docker-compose.yml
```

Upgrade Docker
==
```
update_engine_client -update
```

Run a simple REST service Container
==
docker-compose up -d

Check that registrator has registered the REST service with consul
==
```
core@host-1 ~ $ curl 172.28.128.3:8500/v1/catalog/services | jq
```

Response
===
```
{
 "cesliteralresponsefilter-9110": [],
 "cesliteralresponsefilter-9111": [],
 "consul": [],
 "consul-53": [
               "udp"
              ],
 "consul-8300": [],
 "consul-8301": [
                 "udp"
                ],
 "consul-8302": [
                 "udp"
                ],
 "consul-8400": [],
 "consul-8500": []
}
```

Get some more info from consul about the microservice
==

```
$ curl 172.28.128.3:8500/v1/catalog/service/cesliteralresponsefilter
```

Response
===
```
[
  {
          "Node": "host-1",
          "Address": "172.28.128.3",
          "ServiceID": "host-1:core_annotation_1:9110",
          "ServiceName": "cesliteralresponsefilter-9110",
          "ServiceTags": null,
          "ServiceAddress": "",
          "ServicePort": 9110
  },
  {
          "Node": "host-2",
          "Address": "172.28.128.4",
          "ServiceID": "host-2:core_annotation_1:9110",
          "ServiceName": "cesliteralresponsefilter-9110",
          "ServiceTags": null,
          "ServiceAddress": "",
          "ServicePort": 9110
  }
]
```

Use Consul DNS for the services registered in Consol (DNS loadbalancing)
=

Find the Docker Bridge Interface
==
```
core@host-3 ~ $ export DOCKER_BRIDGE_IP=$(ifconfig docker0 | grep 'inet' | grep -v 'inet6' | awk '{ print [ }')]
```

Spin up another container and ping the service using the consul service name
===

```
core@host-3 ~ $ docker run --dns $DOCKER_BRIDGE_IP --dns 8.8.8.8 --dns-search service.consul --rm --name ping_test -it busybox
```

The DNS response will rotate across the registered services...

Kill a docker service...and the DNS resolution will only use the live services
==


