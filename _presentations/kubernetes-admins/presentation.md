
layout: true
class: middle

---

# Kubernetes for Admins

--

[p.2hog.codes/kubernetes-admins](https://p.2hog.codes/kubernetes-admins)

---


# About 2hog.codes

* Founders of [SourceLair](https://www.sourcelair.com) online IDE + Dimitris Togias
* Docker and DevOps training and consulting

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

# [p.2hog.codes/kubernetes-admins](https://p.2hog.codes/kubernetes-admins)

---

# Agenda

1. Kubernetes intro crash course
1. Setting up a cluster with terraform and kubeadm
1. Inspecting and managing the cluster
1. Ingress and networking in Kubernetes
1. Managing storage for Kubernetes workloads
1. Security principles best practices
1. Managing available resources
1. Adding biases to the orchestrator
1. A quick glimpse of Helm

---
class: center

# Kubernetes intro crash course

---

# Core Kubernetes components

--

* Kubernetes API server
* Controllers
    * Kube controller manager
    * Cloud controller manager
* Networking
* Commonly used plugins

???

* Prometheus
* Dashboard
* (Core)DNS

---

# Node types

--

* Master node(s)
  * The server(s) responsible for meta data storage, API availability and decision making
* Worker nodes
  * The server(s) that are used to run workloads, decided by the management plane

---

# Kubernetes Topology

.center[![:scale 50%](images/kube-architecture.png)]

---

# Kubernetes Topology — a more common example

.center[![:scale 50%](images/kube-architecture-combined.png)]

---

# Kubernetes Topology — a more "cloudy" example

.center[![:scale 50%](images/kube-architecture-cloud.png)]

???

* Story about infrastructure providers to start selling infrastructure in a different way
* Same concept as selling bare metal machines through VMs

---
class: center

# Setting up a cluster with terraform and kubeadm

---

# kubeadm — making Kubernetes cluster bootstrap easy

--

Kubeadm is a tool built to provide `kubeadm init` and `kubeadm join` as best-practice "fast paths" for creating Kubernetes clusters

???

* Making Kubernetes cluster management more close to Swarm
* Making it more secure

--

* Generates a self-signed CA, used to create and verify identities of components in the cluster
* Bootstraps an etcd backend, if an external one is not provided
* Generates default `kubectl` configuration for talking to the cluster

---

# Bootstrapping the cluster

https://github.com/2hog/workshop-infra

--

```bash
# [workshop-vm-XX-00]
sudo kubeadm init
```

???

* This makes the API server be publicly available
* Possibly, we'd like to change this to a private IP, but during the training that's easier if we want to remotely access the cluster
* This has set up a single master node cluster, without any networking

---

# Add the cluster credentials to the `workshop` user

--

```bash
# [workshop-vm-XX-00]
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

# Bootstrap networking

--

* Kubernetes supports any network plugin, as long as it's CNI compatible
* `kubeadm` does not have any network plugin installed, to allow the user to decide one
* We'll go with Weave

???

* Easy to use and deploy
* Fast
* Even supports multicast

---

# Setting up Weave

--

```bash
# [workshop-vm-XX-00]
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
kubectl rollout status daemonset weave-net -n kube-system
```

---

# Joining more nodes to the cluster

--

```bash
# [workshop-vm-XX-01]
sudo kubeadm join IP-HERE:6443 --token TTT --discovery-token-ca-cert-hash sha256:HHH
```

---

# What happened here?

* `kubeadm` connected to the API server, used the token to authenticate and downloaded the information
* Used the CA cert hash to validate that the information is originated from the correct CA

---

# See that everything works

???

* Let's deploy the Kubernetes dashboard
* Let's start inspecting some things there

--

```bash
# [workshop-vm-XX-00]
kubectl apply -f https://gist.githubusercontent.com/akalipetis/968a29bd42b7944f788cb6332b480b62/raw/43c030a7cfd8490b83afd03ce2fb77ee27d1e359/dashboard-rbac.yml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
kubectl -n kube-system rollout status deployment kubernetes-dashboard
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
kubectl proxy
```

---

# Let's check out the dashboard

--

```bash
# [laptop]
ssh -L 8001:127.0.0.1:8001 -C -N workshop@workshop-vm-XX-00.akalipetis.com
```

--

Open http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

---
class: center

# Inspecting and managing the cluster

---

# Get general information about the cluster

--

```bash
# [workshop-vm-XX-00]
kubectl cluster-info
```

---

# Get information about nodes

--

```bash
# [workshop-vm-XX-00]
kubectl get no
kubectl describe node workshop-vm-XX-00
kubectl describe node workshop-vm-XX-01
kubectl descibe nodes
kubectl get componentstatuses
```

---

# See what's running inside the cluster

--

```bash
# [workshop-vm-XX-00]
kubectl get all --all-namespaces
```

---
class: center

# Ingress and networking in Kubernetes

---

# The Kubernetes networking model

--

* Kubernetes supports plugin implementing the CNI
* All pods and nodes in the cluster are routable within a flat network
  * That means that there's no network separation and thus all workloads should be trusted, or resources secured
* Every pod gets an IP
* Pods get DNS names, which work inside the cluster

???

* Sensitive resources should be secured
* There are ways to handle that, but out of topic for this course

---

# Kubernetes load-balancing

--

* Each service in the Kubernetes gets a virtual IP
  * Kubernetes services make sure connections to this internal IP are routed to the correct container, in any host in the cluster
* Multi-host networking is made with CNI plugins
* If desired, services can integrate with external load balancers or simply open a port to each node in the cluster
  * Connections, as soon as they enter the cluster, are routed in the exact same way

???

Benefits:
* You don't need to know where in the cluster every pod runs
* You don't need to do health management

---

# How is DNS resolved in Kubernetes

* Services
    * A record: `my-svc.my-namespace.svc.cluster.local`
        * Multiple A records for "headless" services
    * SRV record: `_my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster.local`
* Pods
    * A record: `pod-ip-address.my-namespace.pod.cluster.local`

---

# Search domains

Except from the full DNS records, you can also search the following DNS names

* Services within the same namespace
  * `svc` will resolve to `svc.the-namespace.cluster.local` when queried from a pod in this namespace
* Services within the same cluster
  * `svc.the-namespace` will resolve to `svc.the-namespace.cluster.local`, even when queried from other namespaces in the same cluster

---

# Exposing services to the world

--

* `LoadBalancer` service type
    * Attaches a cloud provider LB to the service
    * Can either be TCP or HTTP, depending on the cloud provider
* `NodePort` service type
    * Attaches ports (usually 80 and 443) on every node to related ports of the service
    * Is more cloud agnostic
    * It can have a LB on all worker nodes of the cluster in front of it
* `Ingress` API resource

---

# Ingress

--

* A Kubernetes API object that abstracts external access to cluster services
* Does nothing without an ingress controller configured
* Usually implemented using a service of `LoadBalancer` or `NodePort` type with a combination with an ingress controller

---

# Why use Ingress

--

* Easier to expose new services — creating an ingress resource easily automates the process of exposing a service
* Ingress controllers allow for "smarter" things to be done, like path based proxying and more
* Can expose multiple services, using just a Load Balancer, or a set of A/CNAME records

---

# Ingress alternatives

--

* The goods from both worlds, create a "manual" ingress like service and create routes by hand
* Expose every service using a different LB
* Use [sourcelair/ceryx](https://github.com/sourcelair/ceryx) and manually add routes

---
class: center

# Managing storage for Kubernetes workloads

---

# Kubernetes storage related API objects

--

* `PersistentVolume` — an actual, provisioned volume available for consumption
* `PersistentVolumeClaim` — a request to claim a `PersistentVolume` and assign it to a pod
* `StorageClass` — a `PersistentVolume` provisioner, that can dynamically provision `PersistentVolumes` as they are requested

???

* PersistentVolumes and PersistentVolumeClaims are the API that developers interact with admins
* Admins should provide PersistentVolumes inside the cluster, which can be later on claimed by workloads deployed by the developers
* StorageClasses can be configured on the other hand, so that PersistentVolumes can be provisioned dynamically — this is usually related to a cloud provider, or a shared FS like NFS

---

# Storage lifecycle

--

* Birth of a `PersistentVolume`
    * An admin statically creates one
    * One is dynamically created, after a claim could not be serviced and a storage class was provided
* Binding and use
    * The volume is bound, getting access to the `PersistentVolume`
    * The `PersistentVolume` is marked and protected, to avoid accidental deletion and data loss
* Reclaiming unused volumes
    * After the claim is deleted, the volume can be `Retained`
    * Or be `Delete`d
    * The actual behavior is based on the initial provisioning (static VS dynamic) and the reclaim policy

---

# `PersistentVolume` access policies

--

* `RWO` — as soon as a pod claims the volume, it is not available any more
* `ROX` — many pods can have readonly access to the volume
* `RWX` — many pods can have read-write access to the volume

???

* RWO should be used for things like block storage and generally for single-node deployments
* R(W|O)X should be used for volumes that can be mounted to many nodes, like a shared file system

---
class: center

# Security principles best practices

---

# Securing services within the cluster

--

* Services and pods within a cluster are accessible from any pod
* If untrusted pods are run, or if user-facing services exist with lesser access control, services should be secured even if they are internal

---

# Securing services within the cluster

--

* Adding TLS connection support
* Using API keys and other credential types
* Using tools like Istio to control and secure connections between applications

---

# How does Istio work

--

* Injects sidecar containers to every selected pod
* Makes sure all pod traffic is routed from inside the injected Envoy proxy
* It then given knobs to tune the features you want implemented

???

Indicative features:

* Apply access policies
* Secure connections
* Manage traffic
* Telemetry

---

# Istio components

--

* Control plane makes sure the the correct sidecards are injected with the needed settings
* Data plane is consisted of intelligent Envoy proxies, deployed as sidecars

---

# RBAC in Kubernetes

--

* Every pod in Kubernetes gets a service account
* Since the Kubernetes API is routable from every pod, without RBAC the cluster is completely vulnerable

---

# Defaults

* `kubeadm` created a cluster where pods by default do not have access
* The Kubernetes API is secured with HTTPS

---

# RBAC roles in Kubernetes

* Roles define permissions in a Kubernetes cluster
* Permissions for roles are additive (ie whitelist)
* There are two types of Roles in Kubernetes
    * `Role` — defines roles, operating within one namespace
    * `ClusterRole` — defines roles, operating cluster wide

---

# Binding roles to users and service accounts

* `RoleBinding` binds `Role`s to users or service accounts
* `ClusterRoleBinding` binds `ClusterRole`s to users or service accounts
* `ClusterRole` can operate as a `Role` and bound with a `RoleBinding`

---

# Checking out the dashboard `Role` and `RoleBinding`

* https://github.com/kubernetes/dashboard/blob/master/src/deploy/recommended/kubernetes-dashboard.yaml
* https://gist.githubusercontent.com/akalipetis/968a29bd42b7944f788cb6332b480b62/

---

# How service accounts work

* Service accounts create the needed files in `/var/run/secrets/kubernetes.io/serviceaccount/`
* The files created contain the `namespace` and the `token`

--

```bash
# [workshop-vm-XX-00]
kubectl -n kube-system exec weave-net-c9kzt -c weave -it sh
ls /var/run/secrets/kubernetes.io/serviceaccount/
```

---
class: center

# Managing available resources

---

# Kubernetes `requests` vs `limits`

* `requests` — the "lego" size, that the Kubernetes API will use for packing nodes
* `limits` — the actual limit, which will not be surpassed by the container

---

# How are resources measured

* CPU — 1 (1 core), 0.1 (1/10 core) or 150m (150/1000 core)
* RAM — 1024, 1K, or 1Ki are all the same (1 kilobyte)

---

# What if I don't add requests and limits?

--

* Hello `LimitRange`
* Default requests and limits per container
* Can be overriden inside the pod template

---

# `LimitRange` in action

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
EOF
```

---

# Adding quotas to namespaces

* Limit pods that can be created within a namespace
* Limit load balancers and node ports
* Or any other resource

---
class: center

# Adding biases to the orchestrator

---

# Attracting pods to nodes

* Use node affinity to make pods get scheduled to nodes
* Choose between preferred and required based on how mandatory the scheduling decision is

---

# Adding affinity to nodes

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/e2e-az-name
          operator: In
          values:
          - e2e-az1
          - e2e-az2
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 1
      preference:
        matchExpressions:
        - key: another-node-label-key
          operator: In
          values:
          - another-node-label-value
```

---

# Or to other nodes

```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: security
          operator: In
          values:
          - S1
      topologyKey: failure-domain.beta.kubernetes.io/zone
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S2
        topologyKey: kubernetes.io/hostname
```

---

# Repelling pods from nodes

* Taints add a mark on nodes, that make pods get away from them
* They can be no-schedule, prefer no-schedule, or no-execute
* Pods can counter attack taints with tolerations

---

# Example taint

```bash
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key2=value2:NoSchedule
```

--

and toleration

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```

---

# Why `NoSchedule` and `NoExecute`?

* More gradual removal of pods, they mostly get evicted during deployments
* Dynamic tainting of nodes, through the system
    * `node.kubernetes.io/memory-pressure`
    * `node.kubernetes.io/disk-pressure`
    * `node.kubernetes.io/out-of-disk`
    * `node.kubernetes.io/unschedulable`

---
class: center

# A quick glimpse of Helm

---

# What is Helm?

* The package manager for Kubernetes
* Easily parametrize your Kubernetes manifests

---

# Main Helm concepts

* Chart — a package for Helm
* Repository — the chart registry
* Release — an instance of a Chart, running within a Kubernetes cluster

---

# Helm architecture

* Tiller server — a rest API server, which processes and deploys charts
* Helm client — simple CLI to interact with Tiller

---

# Helm templates

* Go template language
* Extensive collection of available functions
* Top level objects

???

Top level objects:
* Release
* Values
* Chart
* Files
* Capabilities
* Template

---

# Installing Helm

```bash
# [workshop-vm-XX-00]
kubectl apply -f https://gist.githubusercontent.com/akalipetis/968a29bd42b7944f788cb6332b480b62/raw/b00ef23c64575605139d8ba4ee1998c3aaa2f88e/tiller-rbac.yml
helm init --service-account tiller
```

---

# Let's deploy our first chart

--

```bash
helm repo update
helm install stable/postgresql
```

---

# Configuring default values

```bash
helm inspect values stable/postgresql > values.yml
vim values.yml
helm upgrade -f=values.yml stable/postgresql
```

---
class: center

# Thanks!
