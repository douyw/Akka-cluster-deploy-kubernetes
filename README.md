
[![Build Status](https://travis-ci.org/IBM/watson-banking-chatbot.svg?branch=master)](https://travis-ci.org/IBM/watson-banking-chatbot)
![IBM Cloud Deployments](https://metrics-tracker.mybluemix.net/stats/527357940ca5e1027fbf945add3b15c4/badge.svg)
<!--Add a new Title and fill in the blanks -->
# Deploy Akka Cluster on Kubernetes
In this Code Pattern, we will deploy a simple Akka application on Kubernetes to demonstrate how you can build an inherently scalable, distributed solution.

Akka is “a set of open-source libraries for designing scalable, resilient systems that span processor cores and networks”. Akka Cluster, in particular, enables the distribution of an application over a cluster, thereby lending elasticity to the application by automatically scaling out or in with the number of members in the cluster. Kubernetes is a container orchestration platform and has the ability to automatically scale the number of pods up or down in response to workload. Together, the two work in concert: your Akka cluster expands automatically as Kubernetes scales the number of pods running for a particular Kubernetes deployment.

One key enabler of this elasticity is the ability for new nodes to join the Akka cluster automatically. Akka clusters are a peer-to-peer framework, however some nodes must be designated “seed” nodes when the cluster is initialized. These serve as an initial point of contact for new nodes. When new nodes are brought online, they can join the cluster automatically by first sending a message to all seed nodes in the cluster, followed by a “join” command to the seed node that responds first. Without seed nodes you would have to manually or programmatically add nodes to the cluster.

This solution uses Docker to build a single container image. In Kubernetes, we use StatefulSet rather than Deployment to guarantee the orderly startup of seed nodes.
## Flow
<!--Remember to dump an image in this path-->
![Architecture Diagram](https://media.github.ibm.com/user/1650/files/3ad7cfd0-161b-11e8-9f5e-3324668e4739)

1. Add tools to the the Docker base image   
2. Use sbt to build the sample app into docker image   
3. Push the docker image to a docker repository
4. Deploy the Akka cluster on Kubernetes with `statfulset`
5. Try out the scaling of Akka cluster

<!--Update this section-->
## Included components
* [Akka](https://akka.io/): A free and open-source toolkit and runtime simplifying the construction of concurrent and distributed applications on the JVM.
* [Kubernetes](https://kubernetes.io): An open-source system for automating deployment, scaling, and management of containers.
* [Docker](https://www.docker.com): Container building,shipping and running platform
* [sbt](http://www.scala-sbt.org/): The interactive build tool for Scala
* [Scala](https://www.scala-lang.org/): A general-purpose programming language providing support for functional programming and a strong static type system.

<!--Update this section when the video is created-->
# Watch the Video


# Steps

<!-- there are MANY updates necessary here, just screenshots where appropriate -->

1. [Clone the repo](#1-clone-the-repo)
2. [Prerequisite](#2-prerequisite)
3. [Build the Docker base image](#3-build-the-docker-base-image)
4. [Build the sample app](#4-build-the-sample-app)
5. [Deploy the cluster on Kubernetes](#5-deploy-the-cluster-on-kubernetes)
6. [Confirm the sample app is working](#6-confirm-the-sample-app-is-working)

### 1. Clone the repo

Clone the `Akka-cluster-deploy-kubernetes` locally. In a terminal, run:

```
git clone https://github.com/IBM/Akka-cluster-deploy-kubernetes
cd Akka-cluster-deploy-kubernetes
```

### 2. Prerequisite

* Create a Kubernetes cluster with either [Minikube](https://kubernetes.io/docs/getting-started-guides/minikube) for local testing, or with [IBM Bluemix Container Service](https://github.com/IBM/container-journey-template) to deploy in cloud. The code here is regularly tested against [Kubernetes Cluster from Bluemix Container Service](https://console.ng.bluemix.net/docs/containers/cs_ov.html#cs_ov) using Travis.
* Ensure the Docker environment variable is configured to point to the registry used by the Kubernetes cluster.
* Have [sbt](https://www.scala-sbt.org/download.html) installed.   

### 3. Build the Docker base image

Build the local Docker base image required by the example app:  

```bash
cat <<EOF | docker build -t local/openjdk-jre-8-bash:latest -
FROM openjdk:8-jre-alpine
RUN apk --no-cache add --update bash coreutils curl
EOF
```

We use `openjdk:8-jre-alpine` as the base image, adding `bash`, networking utility (i.e. `ping`, `telnet`), and `curl` to make it easier for debugging purposes.   


### 4. Build and upload the sample app
Ensure you are at the root directory of the example app. It is optional, but you can modify the Docker repo can be specified in the `build.sbt` file by chaning the `dockerRepository` variable . Sbt will build the docker image with the tag `$repo/myapp:latest`.    
Build the example app by running the following command:

```bash
  sbt docker:publishLocal
```
After the image is built, upload it to docker hub:
```bash
  docker push $repo/myapp:latest
```

### 5. Deploy the cluster on Kubernetes
Modify the `deploy/kubernetes/resources/myapp/myapp-statefulset.json` file to point to your repo if you did changed the repo in step 4. You can find the spec here:
```
"spec": {
        "containers": [
          {
            "name": "myapp",
            "image": "szihai/myapp:latest",
```

Deploy the example app by running the following command. This will create 3 head nodes.

```bash
  kubectl create -f deploy/kubernetes/resources/myapp
```

### 6. Confirm the sample app is working

After a couple minutes, check the 3 head nodes created by the example app (i.e. `myapp-0`, `myapp-1`,`myapp-2`). And let's add a woker node to join the cluster. 

```bash
  kubectl scale statefulsets myapp --replicas=4
```

Once the 4th pod is stood up, we can check the cluster information.

The example app exposes the `/members` endpoint which displays the list of members visible for a particular pod. For example, the following displays the list of members visible from `myapp-0` pod:

```
  kubectl exec -ti myapp-0 -- curl -v myapp-0:9000/members
  *   Trying 172.30.207.9...
  * TCP_NODELAY set
  * Connected to myapp-0 (172.17.0.5) port 9000 (#0)
  > GET /members HTTP/1.1
  > Host: myapp-0:9000
  > User-Agent: curl/7.55.0
  > Accept: */*
  >
  < HTTP/1.1 200 OK
  < Server: akka-http/10.0.10
  < Date: Tue, 12 Dec 2017 04:06:32 GMT
  < Content-Type: application/json
  < Content-Length: 147
  <
  {
  "members" : [ {
    "address" : "akka.tcp://myapp@myapp-0.myapp.default.svc.cluster.local:2551",
    "status" : "Up",
    "roles" : [ ]
  }, {
    "address" : "akka.tcp://myapp@myapp-1.myapp.default.svc.cluster.local:2551",
    "status" : "Up",
    "roles" : [ ]
  }, {
    "address" : "akka.tcp://myapp@myapp-2.myapp.default.svc.cluster.local:2551",
    "status" : "Up",
    "roles" : [ ]
  }, {
    "address" : "akka.tcp://myapp@myapp-3.myapp.default.svc.cluster.local:2551",
    "status" : "Up",
    "roles" : [ ]
  } ]

  * Connection #0 to host myapp-0 left intact
```


# Links

<!-- pick the relevant ones from below -->
# Learn more

* **Kubernetes on IBM Cloud**: Deliver your apps with the combined the power of [Kubernetes and Docker on IBM Cloud](https://www.ibm.com/cloud-computing/bluemix/containers)

<!--keep this-->

# License
This code pattern is licensed under the Apache Software License, Version 2.  Separate third party code objects invoked within this code pattern are licensed by their respective providers pursuant to their own separate licenses. Contributions are subject to the [Developer Certificate of Origin, Version 1.1 (DCO)](https://developercertificate.org/) and the [Apache Software License, Version 2](http://www.apache.org/licenses/LICENSE-2.0.txt).

[Apache Software License (ASL) FAQ](http://www.apache.org/foundation/license-faq.html#WhatDoesItMEAN)
