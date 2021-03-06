
layout: true
class: middle

---
class: center

# Migrating 12 factor applications to Kubernetes


--

[p.2hog.codes/kubernetes-12f-migration](https://p.2hog.codes/kubernetes-12f-migration)

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

# [p.2hog.codes/kubernetes-12f-migration](https://p.2hog.codes/kubernetes-12f-migration)

---

# Agenda

1. Exercise: setting up a Ruby microservice with Sinatra
1. Exercise: setting up a Python microservice with Flask, which needs static assets built with Node.js
1. Exercise: setting up a Python microservice with Django, which connects to the other two services and needs migrations
1. Exercise: setting up a custom microservice
1. Bonus: exposing the Django microservice to the world with Ceryx
1. Case study: from monoliths to a fully-containerized infrastructure

---
class: center

# Exercise: setting up a Ruby microservice with Sinatra

---

# Let's get the code

```bash
git clone https://github.com/2hog/docker-training-samples
cd docker-training-samples
```

---

# Create a Docker image for the microservice

???

* Should install dependencies separately to improve caching

---

# Create a Docker Compose file for local development

???

* Should use an auto-reloading server

---

# Create a Kubernetes deployment manifest

???

* Should also contain an internal service definition

---

# Create a CI/CD file

---

# See it in action

---
class: center

# Exercise: setting up a Python microservice with Flask, which needs static assets built with Node.js

---

# Create a Docker image for the microservice

???

* Should install dependencies separately to improve caching
* Should use multi-stage builds for building and including static assets

---

# Create a Docker Compose file for local development

???

* Should use an auto-reloading server
* Should contain a separate container for the static assets

---

# Create a Kubernetes deployment manifest

???

* Should also contain an internal service definition

---

# Create a CI/CD file

---

# See it in action

---
class: center

# Exercise: setting up a Python microservice with Django, which connects to the other two services and needs migrations

---

# Create a Docker image for the microservice

???

* Should install dependencies separately to improve caching

---

# Create a Docker Compose file for local development

???

* Should use an auto-reloading server
* Should contain the database for easy local development
* Should contain a static (that is pre-built) reference to its depending services

---

# Create a Kubernetes deployment manifest

???

* Should also contain an internal service definition
* Should have environment configuration for discovering the rest of the services
* Should use secrets for the API keys to contact the other services
* Should contain a separate job manifest for running migrations

---

# Create a CI/CD file

???

* Should run migrations using a job

---

# See it in action

---
class: center

# Bonus: exposing the Django microservice to the world with Ceryx

---

# What is Ceryx?

--

* Dynamic proxy, based on NGINX with Lua scripting
* Backed by Redis as a data store
* With a simple API and web UI for configuration

---

# What do we want to deploy

* A Deployment for the proxy
* A Deployment for the API
* A Deployment for the web UI
* A StateulSet for Redis
* A Service of type NodePort (or LoadBalancer, if we were using a cloud provider)

---

# Creating the deployments and services

```bash
kubectl apply -f kube/ceryx/
```

---

# Accessing the API

* We need a way to access the API for Ceryx
* We don't want to `exec` into a pod again

---

# Using `kubectl proxy`

```bash
# [workshop-vm-XX-00]
kubectl proxy

# [local]
ssh -L 8001:127.0.0.1:8001 -C -N workshop@workshop-vm-XX-00.akalipetis.com
# [local]
open http://localhost:8001/api/v1/namespaces/default/services/http:ceryx-api:/proxy/api/routes
```

---

# Let's expose the web UI as a route

```bash
# [local]
curl -XPOST \
  -H 'Content-Type: application/json' \
  http://localhost:8001/api/v1/namespaces/default/services/http:ceryx-api:/proxy/api/routes \
  -d '{
    "source": "ceryx.workshop-vm-XX-00.akalipetis.com",
    "target": "ceryx-web.default.svc.cluster.local"
  }'

open http://ceryx.workshop-vm-XX-00.akalipetis.com
```

---

# Adding routes and previewing them

---
class: center

# Case study: from monoliths to a fully-containerized infrastructure

---

# The stack

--

* Classic Java monoliths
* Deployed to customer servers by people
* Minimal automation, mainly with scripts

---

# The need

--

* A SaaS-like service, serving the same purpose as the customer-deployed products
* Quicker iterations on new versions and features
* Multiple deployed versions of the software at once
* Easy scale up/down of specific components

---

# The answer

--

Docker all the things!

---

# First, baby steps

--

* Put everything in a container
* Create supporting software to make it work as we pleased

---

# Get a PoC ready

--

* Create a container cluster
* Deploy the monolith (in a container)
* Showcase that things can work in containers too

---

# Start splitting

--

* Split parts that are not needed by the new service
* Make logical splits, where things should be separated or scaled differently

---

# Create a continuous integration and deployment system

--

* Make sure tests pass on each push
* Define a branching strategy
* Create a Docker image on master branch and all tags
* Deploy automatically each version that lands in master

---

# Start using the new infrastructure and see how it performs

--

* Monitor all the services for RAM and CPU usage — prometheus to the rescue
* Create nice graphs using Grafana, so that everyone starts to take a glimpse of what is happening
* Try to stabilize the resource usage of the different components

---

# Health check your applications

* Better deployments, since traffic is routed only after a service is healthy
* Kill services that are unhealthy, automatically

---

# Create reservations and enforce limits

--

* After deciding on the optimal memory and CPU usage of each component, start deciding on the reservation and limit to impose
* Use the average with small increase as a reservation, if you don't have a lot of spikes
* Use the (expected) spikes as a hard limit

---

# Plan ahead

--

* Now that you know how each service performs, it's easy to estimate your cluster size
* Start with the reserved memory and CPU, include information from spikes and add some padding for safety
* If you need (small) dynamic scaling, allow for some more space to avoid continuously scale up/down your cluster

---

class: center

# Thanks!
