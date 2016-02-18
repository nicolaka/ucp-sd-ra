## Service Discovery and Load-Balancing with Docker Universal Control Plane (UCP)

### Introduction

When developing modern Docker microservices applications, developers tend to focus more on functionality, speed, robustness, and quality of the application itself more than operational aspects. However, there had been a shift ( aka DevOps ) in application deployment practices that forced developers to own not only the application's development but also its deployment operations (developers are no longer pager-duty-free!). This shift also encouraged the operations teams to provide a common, scalable, and secure infractructure that multiple developer teams can use to build, test, stage and deploy their applications.  

With the new shift, DevOps teams want to ensure that their applications are scalable. This means that these applications need to be broken up into, built, and advertised as smaller, decoupled _microservices_ that can be easily scaled across large compute clusteres.  _microservices_ approach emphasized two key architectural considerations: **service loadbalancing** and **discovery**. This means that as developers build their applications to scale, they need to consider and design how each component ( _service_ ) is being discovered by other services within or from outside the cluster. Additionally, as these services scale horizonatally across the cluster, how can they all be equally utllizied for maximum load distribution. 

Docker Universal Control Plane (UCP) was built with this operational shift in mind. Docker UCP addresses both the developers requirement for seamless path from development to production and the operation's requirement for building a secure and scalable Docker infrastructure. 

### What You Will Learn

In this reference architecture, you will learn how to build and deploy a sample Docker microservice application (Voting App)  with UCP using both built-in service discovery and widely-used loadbalancing solutions. For reference the Docker Compose file of this application is below:

```
version: "2"

services:
  voting-app:
    image: docker/example-voting-app-voting-app
    ports:
      - "80"
    networks:
      - front-tier
      - back-tier
    links:
      - redis

  result-app:
    image: docker/example-voting-app-result-app
    ports:
      - "80"
    links:
      - db
    networks:
      - front-tier
      - back-tier

  worker:
    image: docker/example-voting-app-worker
    networks:
      - back-tier
    links:
      - db
      - redis

  redis:
    image: redis
    ports: ["6379"]
    networks:
      - back-tier
    container_name: "redis"

  db:
    image: postgres:9.4
    volumes:
      - "db-data:/var/lib/postgresql/data"
    networks:
      - back-tier
    container_name: "db"

volumes:
  db-data:

networks:
  front-tier:
  back-tier:
```

### Assumptions

- Basic understanding of Docker Swarm
- Basic understanding of Docker UCP
- Basic understanding of Docker Compose
- Basic understanding of common loadbalancing solutions such as HAProxy or NGINX

### Prerequisites

-  UCP is installed in HA mode ( atleast 3 controllers).
-  UCP nodes are configured ( atleast 2 nodes).
-  Multihost networking is configured on all UCP nodes.

### Requirements

- Docker Compose 1.6.1
- Docker UCP 1.0.0


### Design Considerations

There are multiple considerations for designing production-ready Docker infrastrucutre using UCP. From a developers point of view, it is important to ensure that any design should integrate with the established developer workflow. For example, if develepers use [Docker Compose]() to run their applications locally during development, the new design should ensure Compose files can also be used to deploy to production. Additionally, it's important to ensure that each service  deployed on UCP is easily discoverable and reachable by other services that are part of the same app, regardless where the containers providing these service are deployed in the cluster. This means that developers can assume that moving their apps from local development to production cluster will not break the application. Finally, it is crucial to ensure that the developers' apps are easily discoverable and accessible from outside the cluster ( if neeeded) regardless which cluster or cluster node they end up being deployed to. This means that as the app moves from one cluster to another, developers should not worry about losing accessibility to their applications.

From an operational point of view, it is important to ensure that UCP itself is highly available so that any failure in one or more UCP controllers wouldn't result in downtimes. Additionally, providing a scalable, secure, and stateless load-balancing service for all applications is important to ensure that as application scale, load-balancing can dynamically ensure that traffic is equally distributed across all of the containers providing these services. 

In summary, there are three key design considerations that need to be addressed to ensure the developers and operations' requirements are met

-  UCP High-Avaialibility
-  Internal Service Discovery + Load Distribution
-  External Service Discovery + Load Distribution

### Solution Overview



### 1. UCP High-Availability 

Docker UCP supports high availability (HA) which allows you to replicate your UCP controller along with the underlying Swarm manager and key-value store containers within your cluster. HA requires atleast three (3) controllers, a primary and two replicas , to be configured on three seperate nodes. It is recommonded to never run a cluster with only the primary controller and a single replica as this results in a split-brain scenario. You can easily deploy UCP in HA using a single command by following [these](FIXME) instructions.

UCP controllers are stateless by design. All UCP controllers can handle requests then forward them to the Swarm Master container. Any controller failure when UCP is deployed in HA will not have any impact on your UCP cluster, both from UCP UI access or underlying cluster management perspectives. However, if you're statically mapping a DNS A-Name to a primary UCP controller IP address and the controller itself or the machine goes down, you will not be able to reach UCP. For that reason, it is recommended to deploy a UCP controller loadbalancer. An upstream loadbalancer can distribute all UCP requests to all three controllers behind it. 

[image](image)

**Health Checks**: The loadbalancer can use the UCP API endpoint `/_ping` to ensure that each of the controllers is healthy. A `200 OK`reponse means that the controller is healthy and it can receive traffic. 

**Listeners**:  The loadbalancer should be configured to loadbalance using TCP port 80 and 443 to all three nodes in the cluster. The loadbalancer should not terminate/restablish HTTPS connections. WHY-FIXME

**DNS**: DNS A-name should be mapped to the loadbalancer itself( e.g VIP) and not to any individual controller.

**IPs**: The loadbalancer can loadbalance to the controllers' private or public IPs. 

**SSL Certificates**: When you install UCP, ensure that you use the Fully Qualified Domain Name (FQDN) of your UCP when asked for addtional Subject Alternative Names(SAN). Provide your UCP's FQDN when asked for addtional SAN on **ALL** UCP controllers ( including the replicas ). If you would like to use your own CA to sign UCP's certificate, please follow the following directioners [FIXME]().


Should the primary controller fail, the UCP controller loadbalancer will ensure that UCP can be reached and all Docker deployment workflows are run without impact. This is the case for both UI and CLI traffic.


### 2. Service Discovery + Loadbalancing (Intra-Cluster)

In order for different microservice containers to communicate, they must first discover each other. The introduction of Docker multihost networking in Docker 1.7 ( experimental ) and 1.9 ( official ) had enabled multiservices belonging to a single application to be connected via an [Overlay Network](FIXME) that spans multiple nodes. Docker 1.10 added an embedded DNS-based for hostname lookups for a more reliable and scalable service discovery. Containers deployed using Docker 1.10 can now use DNS to resolve the IP of other containers on the same network. This behaviour works out of the box for both local and overlay user-defined networks.  

Docker 1.10 also introduced the concept of _network alias_. A network alias abstracts multiple containers under a single _service name_. This means that scaled services ( e.g `docker-compose scale service=number`) can be grouped under and resolved by a single alias. Deocker will resolve the alias to a healthy container that belongs to that alias. This is helpful for stateless services in which any container can be used to provide a service. 

**Example** In our sample Voting App, a Java `worker` service can belong to the alias `workers`. When you scale the worker service using Compose, all the `worker` services will belong to a single alias called `workers`. If other services need to connect with any of these services, Docker will resolve the `workers` alias to a healthy container. We can add network aliases for a the `worker` service as follows:


```

<snippet>

  worker:
    image: docker/example-voting-app-worker
	 links:
      - db
      - redis
    networks:
      - back-tier
    network_aliases:
      back-tier:
	    - workers

<snippet>

networks:
  front-tier:
  back-tier:

```

Other services can reach the worker containers by either using the container name (`worker_1` or `worker_1.back-tier`) or the alias name (`workers`).

**Load-Balancing**: Currently the embedded DNS-based service dicovery only pins traffic to a single healthy container that is part of a network alias. There is currently no embedded load balancing that will distribute traffic across all containers within a single alias. If you need to loadbalance traffic within the cluster, you can use the design introduced in the section ( Inter-Cluster Service Discovery). FIXME

### 3. Inter-Cluster Service Discovery + Loadbalancing 

Some services are designed to be accessed from outside the UCP cluster (typically by a DNS name). Whether these services need to be accessed by other services in a different cluster or external public users/services. To access these services, you typically need to creaete a DNS record for each service, and map it to the exact node that that service is running on. If you also need to loadbalance across multiple containers, you need to add a loadbalancer and reconfigure it everytime a container comes up/goes down. This process is tidious and unscalable. 

An easier, more scalable, and automated way of enabling a service discovery and loadbalancing is to have an event-driven service registrator that automatically updates load-balancing backends as new containers come up or go down. 

By combining [Interlock](www.github.com/ehazlett/interlock) with your preferred loadbalancer [HAProxy](https://hub.docker.com/_/haproxy/) or [NGINX](https://hub.docker.com/_/nginx/), you can ensure that your services are externally accessible and load-balanced. 

Interlock is containerized,event-driven tool that connects to UCP's Swarm manager and watches for events. Events can be containers being spun up or going down. It also looks for certain metadata that these containers have. These can be hostnames or labels that you configure the container with. It then uses the metadata to register/de-register these containers as loadbalancing backends. It then ensures that the loadbalancer uses updated backend configs to direct incoming requests to health containers. Both interlock and the loadbalancer containers are stateless, and hence can be scaled horizontally across multiple nodes to provide a highly-available loadbalancing services.


**How it works:**

Interlock and the loadbalancer container need to be deployed on UCP node(s). However, it is recommended to dedicate 1 or more nodes per UCP cluster to provide the external connectivity and loadbalancing service. These nodes need to have externally routable IP addresses. The other nodes running your services do not have to have externally routable IP addresses. In this example we will use a single dedicated UCP node (called **ucp-lb**) to deploy Interlock+LB using Docker Compose.


Once you deploy interlock+lb, you can create a wildcard DNS record that represents your application's domain name and map it to the IP address of the node that's running interlock+lb( in our example it's **ucp-lb**). The metadata added to the containers needs to provide the hostname+domain name for which this container belongs to. That info is then used by interlock to added it to the corresponding loadbalancer backend. Once interlock+lb are deployed, simply adding specific labels to the container will automatically allow external services to reach them.

Interlock uses UCP's k/v store for configs. This enables a single update to the configs to be used by multiple interlock/lb instances ( if you decide to deploy multiple instances of interlock+lb).  

### 3A. Interlock and NGINX/NGINX+

The following steps provide a guideline to configuring the load-balancing solution on a dedicated UCP node  using Interlock+ NGINX Plus:

1. On **any** controller nodes, update Interlock configs using a single curl command against UCP k/v store:

```
$ export CONTROLLER_IP=ucp.enterprise.com
or
$ export CONTROLLER_IP=10.10.10.1

$ docker exec -ti ucp-kv curl \
  --cacert /etc/docker/ssl/ca.pem \
  --cert /etc/docker/ssl/cert.pem \
  --key /etc/docker/ssl/key.pem \
  https://$CONTROLLER_IP:12379/v2/keys/interlock/v1/config -XPUT -d value='listenAddr = ":8080"
dockerURL = "tcp://$CONTROLLER_IP:2376"
tlsCaCert = "/certs/ca.pem"
tlsCert = "/certs/cert.pem"
tlsKey = "/certs/key.pem"

[[Extensions]]
  Name = "nginx"
  ConfigPath = "/etc/conf/nginx.conf"
  PidPath = "/etc/conf/nginx.pid"
  BackendOverrideAddress = ""
  ConnectTimeout = 5000
  ServerTimeout = 10000
  ClientTimeout = 10000
  MaxConn = 1024
  Port = 80
  SyslogAddr = ""
  NginxPlusEnabled = true
  AdminUser = "admin"
  AdminPass = ""
  SSLCertPath = ""
  SSLCert = ""
  SSLPort = 443
  SSLOpts = ""
  User = "www-data"
  WorkerProcesses = 2
  RLimitNoFile = 65535
  ProxyConnectTimeout = 600
  ProxySendTimeout = 600
  ProxyReadTimeout = 600
  SendTimeout = 600
  SSLCiphers = "HIGH:!aNULL:!MD5"
  SSLProtocols = "SSLv3 TLSv1 TLSv1.1 TLSv1.2"'

```
**Note**: Full documentation on Inerlock loadbalancer-specific options can be found [here](https://github.com/ehazlett/interlock/blob/ng/docs/configuration.md).

**NOTE**: (FIXME) If CONTROLLER_IP doesn't get subsitutied by actual name/IP in curl command, edit the command manually and subsitute your local IP of controller.

 
2. On the dedicated UCP node (**ucp-lb**), [install Docker Compose](https://docs.docker.com/compose/install/). 

3. On the dedicated UCP node (**ucp-lb**) , clone the [following gist](https://gist.github.com/nicolaka/ab6e128004af7cd3c7fa). 


4. On the dedicated UCP node (**ucp-lb**), Export an environment variable called **CONTROLLER_IP**. This variable should be the IP address or the DNS name of one of the UCP controllers. **Note**: Make sure to use the DNS/IP that was listed as a SAN when you created your UCP controllers.

	`$ export CONTROLLER_IP=ucp.enterprise.com`

5. On the dedicated UCP node (**ucp-lb**), deploy interlock+nginx using the following docker-compose command:

`$ docker-compose up -d`


6. Confirm that interlock connected to the Swarm event stream:

```
# docker-compose logs
Attaching to nginxlb_nginx_1, nginxlb_interlock_1
interlock_1 | time="2016-02-16T06:20:21Z" level=info msg="interlock 1.0.0 (57c4f86)"
interlock_1 | time="2016-02-16T06:20:21Z" level=debug msg="using kv: addr=etcd://10.10.10.1:12379"
interlock_1 | time="2016-02-16T06:20:21Z" level=debug msg="Trusting certs with subjects: [0\x181\x160\x14\x06\x03U\x04\x03\x13\rSwarm Root CA]"
interlock_1 | time="2016-02-16T06:20:21Z" level=debug msg="configuring TLS for KV"
interlock_1 | time="2016-02-16T06:20:21Z" level=debug msg="using tls for communication with docker"
interlock_1 | time="2016-02-16T06:20:21Z" level=debug msg="docker client: url=tcp://10.0.20.65:2376"
interlock_1 | time="2016-02-16T06:20:21Z" level=debug msg="loading extension: name=nginx configpath=/etc/conf/nginx.conf"
interlock_1 | time="2016-02-16T06:20:21Z" level=debug msg="starting event handling"
```
### 3B. Interlock and HAProxy

The following steps provide a guideline to configuring the load-balancing solution using Interlock+ HAProxy:

1. On **any** controller nodes, update Interlock configs using a single curl command against UCP k/v store:

```

$ export CONTROLLER_IP=ucp.enterprise.com
or
$ export CONTROLLER_IP=10.10.10.1


docker exec -ti ucp-kv curl \
>   --cacert /etc/docker/ssl/ca.pem \
>   --cert /etc/docker/ssl/cert.pem \
>   --key /etc/docker/ssl/key.pem \
>   https://$CONTROLLER_IP:12379/v2/keys/interlock/v1/config -XPUT -d value='listenAddr = ":8080"
> dockerURL = "tcp://$CONTROLLER_IP:2376"
> tlsCaCert = "/certs/ca.pem"
> tlsCert = "/certs/cert.pem"
> tlsKey = "/certs/key.pem"
>
> [[extensions]]
> name = "haproxy"
> configPath = "/usr/local/etc/haproxy/haproxy.cfg"
> pidPath = "/usr/local/etc/haproxy/haproxy.pid"
> sslCert = ""
> maxConn = 1024
> port = 80
> sslPort = 443
> adminUser = "admin"
> adminPass = "interlock"'
```

**Note**: Full documentation on Inerlock loadbalancer-specific options can be found [here](https://github.com/ehazlett/interlock/blob/ng/docs/configuration.md).

**NOTE**: (FIXME) If CONTROLLER_IP doesn't get subsitutied by actual name/IP in curl command, edit the command manually and subsitute your local IP of controller.

2. On the dedicated UCP node (**ucp-lb**), [install Docker Compose](https://docs.docker.com/compose/install/). 

3. On the dedicated UCP node (**ucp-lb**) , clone the [following gist](https://gist.github.com/nicolaka/eb7905b02f3487739284). 

4. On the dedicated UCP node (**ucp-lb**), Export an environment variable called **CONTROLLER_IP**. This variable should be the IP address or the DNS name of one of the UCP controllers. **Note**: Make sure to use the DNS/IP that was listed as a SAN when you created your UCP controllers.

	`$ export CONTROLLER_IP=ucp.enterprise.com`
	
5. On the dedicated UCP node (**ucp-lb**), deploy interlock+haproxy using the following docker-compose command:

`$ docker-compose up -d`

6. Confirm that interlock connected to the Swarm event stream:

```
$ docker-compose up -d
Creating network "haproxylb_default" with the default driver
Creating haproxylb_interlock_1
Creating haproxylb_haproxy_1

$ docker-compose logs
Attaching to haproxylb_haproxy_1, haproxylb_interlock_1
interlock_1 | time="2016-02-16T07:03:34Z" level=info msg="interlock 1.0.0 (57c4f86)"
interlock_1 | time="2016-02-16T07:03:34Z" level=debug msg="using kv: addr=etcd://10.0.20.65:12379"
interlock_1 | time="2016-02-16T07:03:34Z" level=debug msg="Trusting certs with subjects: [0\x181\x160\x14\x06\x03U\x04\x03\x13\rSwarm Root CA]"
interlock_1 | time="2016-02-16T07:03:34Z" level=debug msg="configuring TLS for KV"
interlock_1 | time="2016-02-16T07:03:34Z" level=debug msg="using tls for communication with docker"
interlock_1 | time="2016-02-16T07:03:34Z" level=debug msg="docker client: url=tcp://10.0.20.65:2376"
interlock_1 | time="2016-02-16T07:03:34Z" level=debug msg="loading extension: name=haproxy configpath=/usr/local/etc/haproxy/haproxy.cfg"
interlock_1 | time="2016-02-16T07:03:34Z" level=debug msg="starting event handling"
interlock_1 | time="2016-02-16T07:03:34Z" level=debug msg="checking to reload"
interlock_1 | time="2016-02-16T07:03:34Z" level=debug msg=reloading
interlock_1 | time="2016-02-16T07:03:34Z" level=debug msg="updating load balancers"
interlock_1 | time="2016-02-16T07:03:34Z" level=debug msg="generating proxy config" ext=haproxy
```

Now that Interlock+LB are up and configured to listen on Swarm events. You can start deploying your applications on the UCP cluster. Interlock expects specific container metadata via labels. A complete list of all Interlock options can be found [here](FIXME). 

In our sample app, we can expose two services externally. These services are `voting-app` and `results-app`. For interlock to register these services as backends for the loadbalancer, we need to provide additional container labels. In our case, we want `voting-app` to be registered as `vote.interlock.ucp.dckr.org` and `results-app` as `results.interlock.ucp.dckr.org`. A wildcard DNS record for `*.interlock.ucp.dckr.org` was mapped to the IP addrress of (**ucp-lb**). Interlock will register these services as they come up to ensure that these two services are accessible with the intended DNS names.

Follow the below steps to deploy the app on UCP cluster:

1. Download a UCP client bundle. Instructions can be found [here](FIXME).
2. Ensure that you're pointing your Docker client to the UCP controller:

```
$ cd /path/to/ucp-bundle-admin
ucp-bundle-admin$ source env.sh

$ docker info

```

3. We need to adjust the Compose file to add the necessary Interlock labels. 

For `voting-app` we add the following:


```
    labels:
     interlock.hostname: "vote"
     interlock.domain:   "interlock.ucp.dckr.org"
```


For `results-app` we add the following:

```
    labels:
     interlock.hostname: "results"
     interlock.domain:   "interlock.ucp.dckr.org"
```

The complete docker-compose file now should like this :

```
version: "2"

services:
  voting-app:
    image: docker/example-voting-app-voting-app
    ports:
      - "80"
    networks:
      - front-tier
      - back-tier
    links:
      - redis
    labels:
     interlock.hostname: "vote"
     interlock.domain:   "interlock.ucp.dckr.org"

  result-app:
    image: docker/example-voting-app-result-app
    ports:
      - "80"
    links:
      - db
    networks:
      - front-tier
      - back-tier
    labels:
     interlock.hostname: "results"
     interlock.domain:   "interlock.ucp.dckr.org"

  worker:
    image: docker/example-voting-app-worker
    networks:
      - back-tier
    links:
      - db
      - redis

  redis:
    image: redis
    ports: ["6379"]
    networks:
      - back-tier
    container_name: "redis"

  db:
    image: postgres:9.4
    volumes:
      - "db-data:/var/lib/postgresql/data"
    networks:
      - back-tier
    container_name: "db"

volumes:
  db-data:

networks:
  front-tier:
  back-tier:

```

4. Deploy the app On UCP using Docker Compose:

```
voteapps$ docker-compose up -d

```

5. Access the we app by going to http://vote.interlock.ucp.dckr.org !

6. If you need to scale the voting-app service, you can simply scale it using docker-compose. Interlock will add the newly added container to the voting-app backend.

## Summary

The introduction of UCP had enabled both developers and operations team seamlessly deploy Docker applications on a secure, scalable, and 

