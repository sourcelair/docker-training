layout: true
class: middle

---
class: center

# 12 factor apps in Kubernetes


--

[p.2hog.codes/kubernetes-12f](https://p.2hog.codes/kubernetes-12f)

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

# [p.2hog.codes/kubernetes-12f](https://p.2hog.codes/kubernetes-12f)

---

# Agenda

1. Unifying development, test and production environments
1. From buildpacks to Docker images
1. Deploying 12 factor applications in Kubernetes
1. Configuring applications
1. Sharing secrets and keeping them secret

---
class: center

# Unifying development, test and production environments

---

# The DevOps dream

---

# Getting from here

.center[![:scale 50%](images/before-devops.png)]

--

# To here

.center[![:scale 50%](images/after-devops.png)]

---

# Why have the same environment everywhere?

--

* Avoid works-on-my-machine excuses — it should work everywhere in the same way
* Easily test new features, as the whole stack can run from your local computer, to your CI
* Speed up project onboarding time

---

# Optimizing your delivery pipeline

* Same runtime in development, CI and production
* Use the same declarative format all the way
* Focus on what you do best

???

* Developers should code
* Ops should manage infrastructure
* Application management is left to Kubernetes

---

# Docker as a runtime and image format

???

* Build, release run
* Building Docker images creates an artifact which cannot be changed
* Releasing a Kubernetes manifest, creates a registry in which you can rollback in the future
* The runtime is known and used both locally and in production

--

* Allows for easily distributing a runnable unit in different environments
* Docker containers always run in the same way

---

# The local toolchain

--

* Docker for Mac/Windows
  * Native support for the OS
  * Latest, vanilla Kubernetes distribution
* Docker Compose
* Auto reloading web servers
  * Nodemon, gin, Django, Flask, RoR, etc
* docker-compose up and be ready to go

???

* Alternatively, one could take a look at tools like Stolos or Draft
* Tools like these, allow developing your applications locally and running then in a remote cluster
* This gives the agility of local tooling, while allowing the developer to run the application within a bigger environment

---

# Docker in your CI system

--

* Run your tests in containers using the same image format you use for development and production
* Spin up a test infrastructure in no time and tear it back down
* Do not maintain external testing infrastructure, requiring your build agents to wait in a queue

???

* Imaging having to keep a Postgres database just for testing, there might be collisions or you'll need complex code to handle unique databases
* Cleanup becomes a pain in the ass

---
class: center

# From buildpacks to Docker images

---

# Building efficient Docker images

---

# Base your images to lightweight images

--

* You base images might include a lot of bloat, that you probably do not need
* Your Docker images do not need to include everything you might think of
* You probably do not need a full OS to run your simple process

---

# Let's see an example

--

```bash
docker pull centos
docker pull alpine
docker image inspect centos --format '{{ .Size }}' | awk '{print $1 / 1024 / 1024, "MB"}'
docker image inspect alpine --format '{{ .Size }}' | awk '{print $1 / 1024 / 1024, "MB"}'
```

---

# Keep the same base image for most of your applications

--

* Start with a base image from Docker Hub
* Create your own base image on top of it
* Try to use this as the base image for all your applications

???

* Even if the base image is large enough, basing all images of it allows for quicker pulls and less storage
* Keeping up with security updates is easier if you make sure your base images is always secure
* This can be based on cedar, or any other base image you feel comfortable using

---

# Do not add stuff you do not need

--

* Don't add a debugger, you won't debug your containers
* Don't add cURL

???

* If you need to add a debugging tool to debug a container, you can do it on-demand when you need it
* Adding tools you might not need increases the image size, the attack surface and makes builds slower
* Think of Heroku, you don't have access to any of those things

---

# Group commands to optimize layers

--

* Group commands that do a clean up
* Otherwise, you'll still ship the "large" layer with your image
* Each image layer only _adds up_ space, it doesn't truncate it

---

# Let's see an example

--

```bash
# Dockerfile.split
FROM alpine:latest
RUN apk add -U bind-tools
RUN rm -f /var/cache/apk/*
```

```dockerfile
# Dockerfile.grouped
FROM alpine:latest
RUN apk add -U bind-tools && \
    rm -f /var/cache/apk/*
```

--

```bash
docker build -t split --file=Dockerfile.split .
docker build -t grouped --file=Dockerfile.grouped .
docker image inspect split --format '{{ .Size }}' | awk '{print $1 / 1024 / 1024, "MB"}'
docker image inspect grouped --format '{{ .Size }}' | awk '{print $1 / 1024 / 1024, "MB"}'
```

---

# Optimize cache

--

* Place commands that break the cache further down the Dockerfile
* Place commands that are time-consuming but do not change often at the top

---

# Let's see an example

```dockerfile
# Dockerfile.simple
FROM python:3.6
WORKDIR /code
ADD . /code
RUN pip install -r requirements.txt
```

--

```dockerfile
# Dockerfile.cached
FROM python:3.6
WORKDIR /code
ADD requirements.txt /code/requirements.txt
RUN pip install -r requirements.txt
ADD . /code
```

???

* Although we're creating one more layer, the final image can utilize cache better
* Caching helps a lot during development and CI, as it speeds builds up

---

# Use multi-stage builds

--

* Start from a thin image
* Add all needed dependencies to build your artifact
* Start again, from a smaller image
* Copy your artifact
* Produce a thinner final image

???

* You can think of multi-stage builds, as multiple buildpacks
* Each stage adds a different thing in the game
* Contrary to multiple buildpacks, multi-stage builds do not pass around runtimes

---

# Examples of multiple buildpacks as multi-stage builds

--

* Use the Node.js buildpack to build static assets
* Use Python or Ruby buildpack to package and run your application

---

# More multi-stage build examples

--

* Builds a {JAR, Go/C/C++ binary}, copy it over
* Build a needed library, with lots of dependencies

---

# A multi-stage build example

--

```dockerfile
# Builder buildpack
FROM node:9 as builder
# Install node modules
COPY package.json yarn.lock /build
WORKDIR /build
RUN yarn
# Build static assets
COPY ./static /build/static
RUN yarn run build
# Base buildpack
FROM python:3.6
# Install dependencies
RUN pip install pipenv==2018.7.1
COPY Pipfile* /usr/src/app/
WORKDIR /usr/src/app
RUN pipenv install --deploy --system
# Copy code
COPY ./ /usr/src/app
```

---
class: center

# Configuring applications

---

# How one can configure applications

--

* Use environment variables, the preferred and 12 factor way
* Use ConfigMaps, the bit old-fashioned way
* Bake default configuration in the image at build time, the _shrug_ way

---

# Configuring applications using the environment

--

* Pods can accept environment variables, using the `env` key in their `spec`
* Environment variables can take values from different sources
  * `ConfigMap`s
  * Any fields of the Pod spec — this allows injecting meta data inside the container
  * Secrets

---

# Using ConfigMaps to inject files

--

* `ConfigMap`s can be used to inject values in the environment
* But can be also mounted as volumes (`volumeMounts`) inside the container
    * Be careful though, as volumes override files, there's a tricky syntax to get over that

???

* Though handy, it's not used quite commonly
* HACK: Use `mountPath` and `subPath` /mount/path/file.ext

---

# When to use ConfigMaps

* `ConfigMap`s should be mostly used as a collection of key-value configuration values
* They can be used as the source for environment variables, while they can be created as a group

???

* Pod presets are usually a better alternative — i.e. have a preset for both the worker and web
* Usually, you should not need to mount them all at the same time

---

# Bake default configuration at build time

* This can be done both in code, or in the Dockerfile
* It's not ideal, but it's a good first step if your application cannot work without config
* If your app cannot work with specific files
    * Start by adding them during build in the image
    * Gradually move them in code as defaults and eventually eliminate them

---

# Sharing secrets and keeping them secret

--

* While 12 factor applications did a lot of things correct, secrets in the environment are not ideal
  * Environment can be easily traced or leaked by accident
  * It's not easy to audit that secret values are not leaked

???

* The simplicity of the environment can lead to unwanted leaks
* As the simplest example, error reporting software usually reports the environment for easier debugging

---

# How to share secret values then?

--

* In Kubernetes, secrets are a first class citizen
* The API and use of secrets is exactly the same as with configs
* The difference mainly lies in the _intention_ behind the different objects

---

# How do I even config?

--

_as a rule of thumb..._

* Use sane defaults
* Use environment variables for any configuration that should help in debugging
* Use secrets mounted as volumes for anything that you wouldn't like saved in an external system

---
class: center

# How the rest of the 12 factors apply in Kubernetes

---

# Processes and how they map to pods

--

* Processes map to pods, not to containers
    * Pods do not share any filesystem
    * They can be stateful, but storing state in pods should be avoided
* `StatefulSet`s do not play nicely here, they're an exception
    * That is because Kubernetes can be used to run all kinds of applications, not just 12 factor ones

---

# Port binding

--

* Port binding is abstracted by services
* Each pod binds to a port internally and then services are used as the only communication channel between applications

---

# Concurrency

--

* Concurrency is achieved through system reservations and limits
    * Each pod has a set of resources available to it
    * Each application should be tuned to fit in those boundaries
* If more scale is needed, more replicas should be started
    * This can be either done automatically, or manually

---

# Configuring memory and CPU requests and limits

--

* On every container of a pod, you can defined the `resources` it `requests` and its `limits`
* Requests are reserved within the cluster, used in scheduling decisions
* Limits are enforced, killing containers that try to go past them

???

* By default no limits are set
* Defaults limits can be set though in every namespace, to make it easier to handle default workloads

---

# Requests and limits in action

--

* Use `resources` `limits` and `requests` keys
* Use the `memory` and `cpu` needs and wants of your application

---

# Disposability and why you should not care about pods

--

* Since pods are stateless, you don't really care when do they start and stop
* Even if they're stateful, their state is stored using external volumes
* Services make sure that clients connecting to pods, will always land on the correct one

---

# Build, Release, Run, in Kubernetes

--

* Build with Docker, producing a Docker image
* Release by including this image in a Kubernetes Manifest (ie a deployment)
* Create a new deployment version, by applying the combination of the two

???

* When the image changes, a new version is deployed
* When configuration changes, a new version is deployed
* Pofit

---

# Admin applications in Kubernetes

--

* Run one-off admin commands with jobs
  * Migrations, or other things that should be orchestrated
* Run more debug friendly things, by `exec`ing into pods
  * Check that the DNS resolution works as expected, or directly debug a specific pod

---
class: center

# Thanks!
