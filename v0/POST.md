The best way to learn new technology is through practical implementation, dictated by self-defined mini-goals. Over the next few weeks (hopefully), I intend to learn Kubernetes (and many other things) in an attempt to create a completely Stateless Build Server (and possibly other things).

This post itself will not contain much - just a mission (above) and some intro installation requirements. Ideally, over the next days and weeks, additional blog posts will be released about each revision of this adventure.

## Installation Requirements (mac)

```shell
# Hypervisor
$ brew cask install virtualbox

# kubectl
$ brew install kubernetes-cli

# minikube
$ brew install minikube
```

```shell
# Starting a local k8s cluster in a VM
$ minikube start

# Verify
$ kubectl version
$ kubectl cluster-info
```
