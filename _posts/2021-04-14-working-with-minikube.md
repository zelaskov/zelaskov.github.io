---
title: "working with minikube"
date: 2021-04-14
---

recently i took a look at [minikube](https://minikube.sigs.k8s.io/docs/). at work i use [ecs](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html) and tbh i just simply never worked with k8s so i wanted to give it a try.

good stuff:

* super easy installation and basic setup of minikube and kubectl, at least on OSX
* --driver=hyperkit is so damn fast, i was really surprised by it
* k8s templates are super easy to read/write, especially if u have vsc [extention](https://marketplace.visualstudio.com/items?itemName=ms-kubernetes-tools.vscode-kubernetes-tools) for k8s
* services as an abstract way to expose apps as a network service
* kubectl is super easy to use
* great, detailed and really straightforward documentation.
* there are some great addons, like registry-creds that i used for pulling images from [ECR](https://aws.amazon.com/ecr/)
* huge community
* simple ```minikube dashboard``` gives a cool dashboard with overview of your cluster

cons:
* minikube doesn't have direct access to docker images on your machine - u need to build them *inside* minikube so it can see/use them
* needs a lot of resources
* mostly used for test/dev environments cause you are running single node k8s
