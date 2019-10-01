# Kubespray

## What is kubespray

Kubespray is an Ansible based kubernetes provisioner

It is a set of Ansible roles defined in cluster.yml and tthese roles consists of several play which will install core components required for Kubernetes control plane and other system level components on both nodes required to setup the cluster

It is used for easily deploying production-ready kubernetes clusters

Key features

- Highly available cluster
- Composable (Choice of the network plugin for instance)
- Supports most popular Linux distributions
- Continuous integration tests
- <b>Configurable parameters</b>

## When to use

 - For users who already <b>know Ansible well</b>
 - There is <b>no need to use another tool for provisioning and orchestration</b>


## Compare to others

### kops

Kops enables users to create, update, or delete production-grade and highly available Kubernetes clusters from the <b>command line interface</b>

Compare to kubespray,
 - less flexible in deplyoment patforms.
 - more tightly integrated with the unique features of the clouds
 - can use when you do not wish to use Ansible
 
### Rancher

Rancher is a <b>complete solution</b> for deploying and managing Kubernetes. <br> 
Better than an installer or a platform, it fits perfectly with every part of your container orchestration strategy.

Compare to kubespray,
 - After the clusters are up and running, Rancher manages them with role-based access control (RBAC), deploys workloads onto them, monitors them, notifies you of failures, connects the clusters to your CI/CD system, and gives you a complete solution for using Kubernetes

 - Kubespray exist to deploy Kubernetes clusters but can only work with a handful of providers. Both focus on deploying infrastructure and then installing Kubernetes, and then they leave you to figure out what to do next. If something fails during the deploy, their complexity makes it difficult to debug what went wrong

refers to [here][rancher link]

## Useful web ui 

Research useful web ui opensources for kubernetes after deploying with kubespray

- ### [kube-web-view]
- ### [kubernator]
- ### [k8dash]


## Getting-started
refers to [here][getting-started]


## Official Pages
### https://kubespray.io

## References

https://kubespray.io

https://srcco.de/posts/kubernetes-web-uis-in-2019.html

https://rancher.com/what-is-rancher/overview

https://dzone.com/articles/kubespray-10-simple-steps-for-installing-a-product


[rancher link]: https://rancher.com/what-is-rancher/how-is-rancher-different/
[getting-started]: 03-getting-started.md
[kube-web-view]: https://codeberg.org/hjacobs/kube-web-view/
[kubernator]: https://github.com/smpio/kubernator
[k8dash]: https://github.com/herbrandson/k8dash
