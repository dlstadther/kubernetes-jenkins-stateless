# Jenkins/K8s

Learning Kubernetes through Jenkins deployments and configurations

## Goal
Create a completely stateless build server

## Revisions
* v1: proof of concept; following blog post; nothing original
* v2: v1 + refactor; combine into single K8s recipe; add ServiceAccount
* v3: v2 + JCasC Plugin; add Role; add ConfigMap; resolve Jenkins security warnings
* v4: v3 + K8s executors; utilize K8s DNS; add non-default namespace

## Resources
* General
    * https://kubernetes.io/
    * https://akomljen.com/
    * https://kumorilabs.com/categories/kubernetes/
    * https://medium.com/@tsuyoshiushio/kubernetes-in-three-diagrams-6aba8432541c
* Prometheus
    * [Alen Komljen - 2018-04-15](https://akomljen.com/get-kubernetes-cluster-metrics-with-prometheus-in-5-minutes/)
* Jenkins
    * [Yuri Bushnev - 2018-03-05](https://www.blazemeter.com/blog/how-to-setup-scalable-jenkins-on-top-of-a-kubernetes-cluster)
* Jenkins (using Helm)
    * [Alen Komljen - 2018-02-18](https://akomljen.com/set-up-a-jenkins-ci-cd-pipeline-with-kubernetes/)
* Jenkins (persistentVolume)
    * https://kumorilabs.com/blog/k8s-6-integrating-jenkins-kubernetes/
