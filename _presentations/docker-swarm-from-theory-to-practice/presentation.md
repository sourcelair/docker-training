layout: true
class: middle

---

# Docker Swarm: from theory to practice

---

# .center[2hog]

.center[We teach the lessons we have learnt the hard way in production.]

.footnote[https://2hog.codes]

--

.center[Consulting, training and contracting services on containers, APIs and infrastructure]

---

# Antonis Kalipetis

* Docker Captain and Docker Certified Associate
* Python lover and developer
* Technology lead at SourceLair, Private Company

.footnote[[@akalipetis](https://twitter.com/akalipetis)]

---

# Paris Kasidiaris

* Python lover and developer
* CEO at SourceLair, Private Company
* Docker training and consulting

.footnote[[@pariskasid](https://twitter.com/pariskasid)]

---

# Dimitris Togias

* Self-luminous, minimalist engineer
* Co-founder of Warply and Niobium Labs
* Previously, Mobile Engineer and Craftsman at Skroutz

.footnote[[@demo9](https://twitter.com/demo9)]

---

class: center

# [p.2hog.codes/docker-swarm-from-theory-to-practice](https://p.2hog.codes/docker-swarm-from-theory-to-practice)

---

# Agenda

1. What is a container, a crash course
1. From servers to clusters: intro to containerized infrastructure
1. Docker swarm in practice, setting up and managing a cluster
1. Deploying services to Docker Swarm
1. Production deployments in a containerized infrastructure
1. Swarm debugging
1. Tech chat + coffee

---
class: center

# What is a container, a crash course

---

# What is a Container?

Containers are a set of **kernel tools and features** that **jail** and **limit** a **process** based on our needs.

---

# Virtual Machines vs. Containers?

They should co-exist. We should run N Containers in M Virtual Machines (N > M).

Imagine a Virtual Machine as a multi-floor building and a Container as a rented flat.

- **Virtual Machines** provide deep isolation, so they are heavy and not versatile
- **Containers** are fast and lightweight

???
* They share the same plumbing
* Each flat has its own limits
* They all must cooperate for the good operation of the building

VS

* You have your own pool
* You can turn up the heat whenever you want
* Comes at a cost
* Fixing infrastructure issues is more time and money consuming

---

# What is a Container? (in a bit more details)

* It’s a process
* Isolated in it’s own world, using **namespaces**
* With limited resources, using **cgroups**

---

# Namespaces

A **namespace** wraps a global system resource in an abstraction that makes it appear to the processes within the namespace that **they have their own isolated instance of the global resource**. Changes to the global resource are visible to other processes that are members of the namespace, but are invisible to other processes. One use of namespaces is to **implement containers**.

.footnote[The Linux man-pages project:<br />http://man7.org/linux/man-pages/man7/namespaces.7.html]

---

# Popular Namespaces

* net
* mnt
* user
* pid

---

# cgroups

**cgroups** (abbreviated from control groups) is a Linux kernel feature that **limits, accounts for, and isolates** the resource **usage** (CPU, memory, disk I/O, network, etc.) of a collection of processes.

.footnote[Wikipedia:<br />https://en.wikipedia.org/wiki/Cgroups]

---

# Popular cgroups

* memory
* cpu/cpuset
* devices
* blkio
* network*

.footnote[*network is not a real cgroup, it’s used though for metering]

---

# Containers and images

* Images are checkpoints of the file system, from which we start containers
* They are layered, using a copy on write file system
* We can start containers from images and create images from containers in a specific point in time

???

* Imagine layers as paintings
* I can start from the same painting
* Each time, I can add a transparent layer on top and draw my changes to the painting
* The initial painting and the layer have all the information
* The footprint of the new painting is just the layer

---

## Imagine the following pieces, as different image layers

* Ubuntu
* Python 3.6
* Dependencies
* Application code
* Base configuration

???

All images start from `scratch` and build on top of it

---

# Dockerfiles, the recipes for images

* Start from a base image
* Follow a set of reproducible instructions
  * Run commands
  * Add code
  * Define meta-data
* After running them, a new image is created

---

# Managing state

* Containers are ephemeral
* Restarting a container, does not persist its state

--

## Managing state with volumes

* Volumes could be as simple as a directory on the host
* ...or as complex as a block storage device
* Containers can have volumes mounted, to save persistent state in them

---

# Where's my localhost?

* Each container gets its own virtual ethernet
* Container cannot talk with each other, or the host, using localhost
* We need a way to network containers

--

## Mutli-container networks

* All containers get their unique IP within a network
* Service discovery is done through DNS, using the internal DNS server
* Each container can be part of multiple networks

???

* Networks are software defined

---
class: center

# From servers to clusters: intro to containerized infrastructure

???

* We talked about running Docker containers in a single node, but we need more for production
* Swarm takes care of managing your whole cluster
* You don't have to know the health of each node any more
* Start knowing the health of your services

---

# Key concepts

* Nodes
  * The servers in your cluster, either managers or workers
* Services
  * A group of tasks that share the same configuration
* Tasks
  * Mapped 1-1 with containers, but could be any deployable unit

---

# Topology Docker Engine

.center[![:scale 50%](/images/docker-swarm-from-theory-to-practice/docker-engine-topology.png)]

---

# Swarm roles

---

# Swarm roles

.left-column[
## Managers
]

.right-column[
* Use Raft Consensus Algorithm
* Are responsible for applying the declarative state in the cluster
* M-TLS, encrypted on operation
  * Optionally encrypted at rest
* Should direct to them in order to manage the cluster
* Can run payload, but should be avoided in bigger clusters
]

---

# Swarm roles

.left-column[
## Managers
## Workers
]

.right-column[
* Only run payload within a cluster, when directed by managers
* Are responsible for applying the declarative state in the cluster
* mTLS communication with managers
* Wait for payload to be pushed to them
* Only have access to the resources assigned to them
]

???

* Have cryptographic identity
* Minimal threat-model, as they do not have access to things like secrets if they're not assigned to them
* Only have access to the payload sent to them, aka registry secrets, etc

---

# Topology of Docker Swarm

.center[![:scale 80%](/images/docker-swarm-from-theory-to-practice/swarm-topology.png)]

---

# Declarative vs imperative infrastructure

## Say what you want, not how to achieve this

---

# A simple example

* _Declarative:_ I want a coffee, sweet, without milk
--

  * ...never drink sweet coffee, but ¯\\_(ツ)_/¯
--


VS

--

* _Imperative:_ I want you to take the beans, ground them, boil water and then mix them. Then, add some sugar and give it to me.

---

# A bit more on declarative infrastructure

* It's easier for the user, as long as the software is logical
* All the state is saved within the cluster
* The cluster continuously tries to make sure that the declared state and the current state of the cluster are a match

---

# Docker Swarm has security in its DNA

* Least privilege architecture, using the push model
* Every node within the cluster has a cryptographic identity
* All communications are encrypted, using mTLS
* Option for everything to be encrypted at rest

???

* The managers push the information that is needed, only when it's needed, to the worker nodes that need it
* Certificates are rotated automatically, without producing cluster breakage
* Mutual TLS is used both for encrypting communications, but also for authentication and authorization
* Nodes can either save the key in disk, or request it when booting to be unlocked

---
class: center

# Docker swarm in practice, setting up and managing a cluster

---

# Creating a new Swarm

```bash
[workshop-vm-XX-1]
docker swarm init --advertise-addr=eth0
```

---

# What just happened?

* a keypair is created for the root CA of our Swarm
* a keypair is created for the first node
* a certificate is issued for this node
* the join tokens are created

???

* The CA is used to generate keys for every node and verify their identity
* Join tokens include the role of the node that will be joined and are rotated when all keys are rotated

---

# Inspecting the cluster

```bash
[workshop-vm-XX-1]
docker node ls
```

--
```bash
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
8z8uqavgsyupbs0t3ormc84ya *   workshop-vm-XX-1    Ready               Active              Leader              18.03.0-ce
```

???

Currently, there's just one node in the cluster, which is a manager and a leader.

---

# Join another node in the Swarm as worker

```bash
[workshop-vm-XX-2]
docker swarm join --token SWMTKN-1-WWW 10.0.74.3:2377
```

---

# Join another node in the Swarm as manager

```bash
[workshop-vm-XX-2]
docker swarm join-token manager
```
--
```bash
[workshop-vm-XX-1]
docker swarm join-token manager
```

???

* They can try to get the token from a worker, which will not work
* Help them understand the importance of the manager

--

```bash
[workshop-vm-XX-3]
docker swarm join --token SWMTKN-1-MMM  10.0.74.3:2377
```

---

# List the nodes in your cluster

```bash
[workshop-vm-XX-3]
docker node ls
```

--

```bash
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
8z8uqavgsyupbs0t3ormc84ya *   workshop-vm-XX-1    Ready               Active              Leader              18.03.0-ce
2wowq8ms2lqkixzz9mzzjx0je     workshop-vm-XX-2    Ready               Active                                  18.03.0-ce
v20fuo5g8lnfy96sr19315166     workshop-vm-XX-3    Ready               Active              Reachable           18.03.0-ce
```

---

# Checking the different information

```bash
[workshop-vm-XX-1]
docker info
```

--

```bash
[workshop-vm-XX-2]
docker info
```

---

# Let's play a game, kill the daemon in one node

```bash
[workshop-vm-XX-3]
sudo systemctl stop docker
```
--

```bash
[workshop-vm-XX-1]
docker node ls
```

--

```bash
Error response from daemon: rpc error: code = Unknown desc = The swarm does not have a leader. It's possible that too few managers are online. Make sure more than half of the managers are online.
```

???

* 2 managers is a bad idea, you always need N / 2 + 1 for consensus
* let's increase the managers

---

# Let's restore the node

```bash
[workshop-vm-XX-3]
sudo systemctl start docker
```

---

# Let's promote workshop-vm-XX-2 to manager

```bash
[workshop-vm-XX-3]
docker node promote workshop-vm-XX-1
```

--

```bash
[workshop-vm-XX-3]
docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
vz22xh8zj1xjouflm9rlnfhxu     workshop-vm-XX-1               Ready               Active              Leader
ec0eypoz0o6btv77uyjqux1l9     workshop-vm-XX-2               Ready               Active              Reachable
hguvzetdq838qseegng6n58ow *   workshop-vm-XX-3               Ready               Active              Reachable
```

---

# Let's destroy everything, again

```bash
[workshop-vm-XX-3]
sudo systemctl stop docker
```

--

## or not...

--

```bash
[workshop-vm-XX-1]
docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
vz22xh8zj1xjouflm9rlnfhxu *   workshop-vm-XX-1               Ready               Active              Leader
ec0eypoz0o6btv77uyjqux1l9     workshop-vm-XX-2               Ready               Active              Reachable
hguvzetdq838qseegng6n58ow     workshop-vm-XX-3               Ready               Active              Unreachable
```

???

* Now even though we have 2 managers, even if one goes down the cluster is unmanageable
* We need at least 3 to make sure we have HA

---

# Distributed systems need consensus

* Consensus involves multiple servers agreeing on values
* Typical consensus algorithms make progress when any majority of their servers is available
  * this means that at least N/2+1 members should be available for the cluster to be operational
  * for example, a cluster of 5 servers can continue to operate even if 2 servers fail

---

# Why should I care?

--

* Behind the scenes, Docker Swarm is using a distributed KV store, based on RAFT
* RAFT is a consensus algorithm, also used in other distributed systems like ETCD
* Docker Swarm clusters should always have an odd number of managers

---
exclude: true

.center[<iframe src="https://raft.github.io/raftscope/index.html" style="border: 0; width: 800px; height: 580px; margin-bottom: 20px"></iframe>]

---

# Encryption at rest

--

* Docker Swarm supports encryption at rest, but we have to enable it
* We need to unlock the Swarm when it's restarted using a key

---

# Enabling auto-lock

```bash
[workshop-vm-XX-1]
docker swarm update --autolock=true
```

--

```bash
Swarm updated.
To unlock a swarm manager after it restarts, run the `docker swarm unlock`
command and provide the following key:

    SWMKEY-1-ZZZ

Please remember to store this key in a password manager, since without it you
will not be able to restart the manager.
```

???

* This key is needed to unlock the node when it boots
* If we lose the key and all manager shut down, we doomed!
* Keep this in a safe place

---

# Let's lock a node

```bash
[workshop-vm-XX-1]
# Lock the node, just by stopping the daemon
sudo systemctl stop docker
```

--

```bash
# Verify that the node is locked after restart
sudo systemctl start docker
docker node ls
```

--


```bash
Error response from daemon: Swarm is encrypted and needs to be unlocked before it can be used. Please use "docker swarm unlock" to unlock it.
```

---

# ... and unlock it

```bash
[workshop-vm-XX-1]
docker swarm unlock
```

--

## Remember that to unlock a node, you need the key

---

# In a nutshell

* Removing auto-lock adds a greater layer of security
* ...by reducing automation

???

* This will be a pain in 3-node clusters, but could be easier to handle in larger clusters, which have 5+ managers
* If more than half of the nodes are locked, the cluster cannot operate

---
class: center

# Deploying services to Docker Swarm

---

# What is a service?

* Services are the definition of the state that we want the cluster to move in
* From services, tasks are created, which map 1-1 with containers

???

* Services are the desired state, the coffee
* Tasks are the realization of this desired state, they are created so that they match

--

```bash
[workshop-vm-XX-1]
docker service create --publish=8080:80 --name=nginx nginx:latest
```

## Let's check this out

???

* No matter which node you are in, the port opens
* This is because Docker Swarm have ingress load balancing

--

```bash
[workshop-vm-XX-1]
curl localhost:8080
```

???

You can also open the external IP, port of a node

--

```bash
[workshop-vm-XX-1]
docker ps
```

---

# Where are my containers?

* When a service gets created, we instruct Swarm to always have at least N number of containers running
* Swarm has a scheduler built-in, which decides where that container is going to run
* We can help the orchestrator decide where to place each container, if we have constraints though

---

# What's a scheduler

* A tetris player
* Always running a for loop
* Trying to reach our _declared_ state

---

# Docker load-balancing

* Each service in the Swarm gets a virtual IP
  * The Swarm makes sure connections to this internal IP are routed to the correct container, in any host in the Swarm
* Multi-host networking is made with pluggable network drivers
* If desired, a port is opened to each node of the Swarm
  * Connections to this port are routed to the service virtual IP, at a defined port

???

Benefits:
* You don't need to know where in the cluster every service runs
* You don't need to do health management

---

# Updating the service

```bash
[workshop-vm-XX-1]
docker service update --image=nginx:does-not-exists nginx
```

--

```bash
image nginx:does-not-exists could not be accessed on a registry to record
its digest. Each node will access nginx:does-not-exists independently,
possibly leading to different nodes running different
versions of the image.

nginx
overall progress: 0 out of 1 tasks
1/1: No such image: nginx:does-not-exists
service update paused: update paused due to failure or early termination of task a01zttpvvavwsedgx6lerw2xc
```

--

```bash
[workshop-vm-XX-1]
docker service ps nginx
ID                  NAME                IMAGE                   NODE                DESIRED STATE       CURRENT STATE             ERROR                           PORTS
z38051h4kf4t        nginx.1             nginx:does-not-exists   workshop-vm-XX-2               Ready               Rejected 4 seconds ago    "No such image: nginx:does-not…"
yadhc69tuauz         \_ nginx.1         nginx:does-not-exists   workshop-vm-XX-1               Shutdown            Rejected 9 seconds ago    "No such image: nginx:does-not…"
xwxtx6fc5hmp         \_ nginx.1         nginx:does-not-exists   workshop-vm-XX-1               Shutdown            Rejected 13 seconds ago   "No such image: nginx:does-not…"
ov5i7c5y71zc         \_ nginx.1         nginx:latest            workshop-vm-XX-1               Shutdown            Shutdown 9 seconds ago
```

---

# How to get to a previous state

* Swarm keeps the state in RAFT
* For each service, it also keeps the previously deployed state
* Let's head a state back!

---

# Rolling back

```bash
[workshop-vm-XX-1]
docker service rollback nginx
```

--

```bash
[workshop-vm-XX-1]
docker service ps nginx
ID                  NAME                IMAGE                   NODE                DESIRED STATE       CURRENT STATE                 ERROR                              PORTS
hak9ijaaubgi        nginx.1             nginx:latest            workshop-vm-XX-1               Running             Running 11 seconds ago
yylxs4ktx0ab         \_ nginx.1         nginx:does-not-exists   workshop-vm-XX-2               Shutdown            Rejected 49 seconds ago       "No such image: nginx:does-not…"
yy1z6lb6rqsm         \_ nginx.1         nginx:does-not-exists   workshop-vm-XX-1               Shutdown            Rejected about a minute ago   "No such image: nginx:does-not…"
z38051h4kf4t         \_ nginx.1         nginx:does-not-exists   workshop-vm-XX-2               Shutdown            Rejected about a minute ago   "No such image: nginx:does-not…"
yadhc69tuauz         \_ nginx.1         nginx:does-not-exists   workshop-vm-XX-1               Shutdown            Rejected about a minute ago   "No such image: nginx:does-not…"
[workshop-vm-XX-1]
```

---

# Securely storing secrets

* Manage sensitive data within containers
* Database passwords, SSH keys, TLS certificates

---

# Secrets security
* Cryptographically stored inside the Raft log
* Mounted as an in-memory filesystem to the container
  * Available only to the node that the payload runs
  * Cannot be found in the disk
  * Seen as file within the container
* Secrets cannot be retrieved through the API

???

* Environment variables leak
* Minimum attack vector

---

# Let's see this in an example

```bash
[workshop-vm-XX-1]
docker secret create some-sec -
```

???

* Create a new secret using stdin

--

```bash
[workshop-vm-XX-1]
docker service create \
  --secret=some-sec \
  --workdir=/run/secrets \
  --publish=8888:8000 \
  python:3.6 \
  python -m http.server 8000
```

???

* Curl the file now

--

```bash
[workshop-vm-XX-1]
curl localhost:8888/some-sec
Don't tell this to anyone
```

---

# Configuring services

* Docker configs allow you to store configuration inside the Swarm cluster
* Like secrets, but can be retrieved and mounted to any path

---

# Create a new config


```bash
[workshop-vm-XX-1]
cat << EOF > index.html
<html>
  <head><title>Hello Docker</title></head>
  <body>
    <p>Hello Docker! You have deployed an HTML page.</p>
  </body>
</html>
EOF
```

--

```bash
[workshop-vm-XX-1]
docker config create homepage index.html
```

--

```bash
[workshop-vm-XX-1]
docker service create \
  --config=source=homepage,target=/usr/share/nginx/html/index.html \
  --publish=9999:80 \
  nginx
```

---
class: center

# Production deployments in a containerized infrastructure

---

# From services to stacks

Stacks are logical collections of services, which define complete applications

---

# Stacks in Docker Swarm

* They are compose files
* They are just a client abstraction, using labels, not a server resource
* They rock!

---

# Deploying stacks

```bash
[workshop-vm-XX-1]
# First, let's create a compose file
cat << EOF > stack.yml
version: '3.5'
services:
  nginx:
    image: nginx:alpine
    ports:
      - 9090:80
EOF
```

--

```bash
# ...and now, let's deploy it
docker stack deploy -c stack.yml my-stack
```

---

# Let's do something more interesting

```bash
[workshop-vm-XX-1]
# Update the compose file, to have a different image
cat << EOF > stack.yml
version: '3.5'
services:
  nginx:
    image: akalipetis/flask-hostname:latest
    command: flask run --host=0.0.0.0
    environment:
      FLASK_APP: app.py
    ports:
      - 9090:5000
EOF
docker stack deploy -c stack.yml my-stack
```

---

# Let's see the load-balancing in action

Scaling a service

--

```bash
[workshop-vm-XX-1]
docker service scale my-stack_nginx=10
```

--

```bash
# Does it really work?
for i in $(seq 1 20); do curl localhost:9090; echo; done
```

--

```bash
docker ps -f label=com.docker.swarm.service.name=my-stack_nginx
```

???

* Services were distributed throughout the cluster
* Requests were load balanced to all services, not depending on the node that was hit

---

# Separating a cluster in groups

There are times that not every payload should be deployed to every node

* a service needs an SSD disk
* there's a special CPU somewhere
* the node needs to be PCI-DSS certified
* a service should span multiple availability zones

---

# Let's first start by labeling our nodes

```bash
[workshop-vm-XX-1]
# Add PCI label to nodes 1 and 2
docker node update --label-add=com.example.pci=true workshop-vm-XX-1
docker node update --label-add=com.example.pci=true workshop-vm-XX-2
# Add node 1 to AZ A and nodes 2 and 3 to AZ B
docker node update --label-add=com.example.az=a workshop-vm-XX-1
docker node update --label-add=com.example.az=b workshop-vm-XX-2
docker node update --label-add=com.example.az=b workshop-vm-XX-3
```

---

# Let's deploy a constrained service

```bash
[workshop-vm-XX-1]
docker service create \
  --constraint=node.labels.com.example.pci==true \
  --replicas=6 \
  --name=constrained \
  nginx
```

--

```bash
[workshop-vm-XX-1]
# Check what we did
docker service ps constrained
```

---

# We can't satisfy everyone

```bash
[workshop-vm-XX-1]
docker service create \
  --constraint=node.labels.com.example.unsatisfied==true \
  --name=unsatisfied \
  nginx
```

--

```bash
[workshop-vm-XX-3]
# Let's fix This
docker node update --label-add=com.example.unsatisfied=true workshop-vm-XX-3
```

---

# Let's deploy a constrained service

* The service was only deployed to nodes satisfying the constraint
* This was true, even when the constraint was not satisfied by any node
* When labels change, unscheduled payload gets scheduled, if its constraints are met
* ...or scheduled payload might be evicted

???

* The service has only beed deployed on the nodes that satisfy the constraint
* Every time a label is changed, the Swarm checks if there are payloads that can either be scheduled, or should be evicted

---

# Let's see what happens when we set memory reservation

```bash
[workshop-vm-XX-1]
docker service create \
  --reserve-memory=4G \
  --name=mem-hungry \
  nginx
```

???

Everything is good, but what happens after we scale this?

--

```bash
# Let's scale this a bit
[workshop-vm-XX-1]
docker service scale mem-hungry=7
```

---

# Let's see what happens when we set memory reservation

* The service was successfully deployed, as the available memory was more than the reserved one
* When we scaled, tasks were successfully scheduled, until the capacity of the nodes was reached

---

# Let's spread things around availability zones

```bash
[workshop-vm-XX-1]
docker service create \
  --placement-pref=spread=node.labels.com.example.az \
  --replicas=6 \
  --name=spread \
  nginx
```

--

```bash
[workshop-vm-XX-1]
docker service ps spread
```

???

* The service was evenly distributed between different availability zones
* Since workshop-vm-XX-1 was the only one in its zone, it got half the services

---

# Let's spread things around availability zones

* Services can be spread among the nodes with different values
* A common pitfall here, is that missing labels are considered as a null value

---

# How does Swarm schedule things

* finds the nodes that satisfy the constraints of a service
  * label constraints
  * architecture/OS constraints
  * resource (RAM/CPU) needs and availability
* spreads the tasks to nodes, taking into account
  * placement preferences
  * node load, trying to distribute load in the best way

---
class: center

# Swarm debugging

---

# Recovering from failed RAFT state

* the RAFT log might get corrupted
* catastrophic failure of more than half of the nodes

---

# Debugging service failures

Reading logs is usually the first thing to do

--

```bash
[workshop-vm-XX-1]
# Read the logs of all tasks of service
docker service logs my-stack_nginx
```

--

```bash
[workshop-vm-XX-1]
# ...or just a single tasks
docker service logs kxqxntzly0bn
```

???

* Get the task ID from the previous logs
* Alternatively, use docker service ps ...
* And of course, there are many ways to query specific dates/times

---

# Checking DNS and connectivity

```bash
[workshop-vm-XX-1]
# Jump into a container
docker exec -it \
  $(docker ps -q -f label=com.docker.swarm.service.name=my-stack_nginx | head -n 1) \
  sh
# Check the DNS records for the service and the tasks
dig nginx
dig tasks.nginx
# For multiple networks, use the network namespaced DNS
dig nginx.my-stack_default
dig tasks.nginx.my-stack_default
```

---

# Checking services one by one

Using the IPs previously acquired

--

```bash
apk add -U curl
curl 10.0.0.12:5000
```

---

# Recovering from failed RAFT state

```bash
[workshop-vm-XX-1]
# Stop the Docker daemon
sudo systemctl stop docker
# Back up the Swarm directory
sudo cp -r /var/lib/docker/swarm /swarm.back
# Restart the Docker daemon
sudo systemctl start docker
# Force re-initialize the cluster
docker swarm init --force-new-cluster
```

--

Then we need to rejoin nodes and remove the old, not obsolete ones

---

# Monitoring our cluster

* Prometheus is the hottest monitoring solution
  * it has native support for Docker Swarm services
  * there are great examples in the internets to get up and running quickly
* there are other tools, Like
  * the ELK stack
  * DataDog
  * lots more...

.footnote[[stefanprodan/swarmprom](https://github.com/stefanprodan/swarmprom)]

---
class: center

# Thanks!
