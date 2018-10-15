There's no better way to learn a new technology than through a userful example. [Jenkins](https://jenkins.io/) has the ability to be integrated with [Kubernetes (K8s)](https://kubernetes.io/) to enable dynamic creation of executor Pods. The following content traces my day through learning K8s by deployment a super-charged, flexible Jenkins build server.

## Install Requirements (Mac)
```shell
# Hypervisor
$ brew cask install virtualbox

# KubeCtl
$ brew install kubernetes-cli

# Minikube
$ brew install minikube
```

```shell
# Starting a local k8s cluster in a VM
$ minikube start

# Verify
$ kubectl version
$ kubectl cluster-info
```


# Jenkins Pod

For a non-trivial example, I wanted to deploy scalable Jenkins within Kubernetes on MiniKube. I followed [this blog post (2018-03-05)](https://www.blazemeter.com/blog/how-to-setup-scalable-jenkins-on-top-of-a-kubernetes-cluster).


## Jenkins Master Installation

Let's begin with a local directory to keep our build materials

```shell
$ mkdir -p ~/tutorial/k8s/jenkins
```

Now to define our Dockerfile for Jenkins.

```shell
$ cat > ~/tutorial/k8s/jenkins/Dockerfile <<EOF
from jenkins/jenkins:2.143

# Distributed Builds plugins
RUN /usr/local/bin/install-plugins.sh ssh-slaves

# install Notifications and Publishing plugins
RUN /usr/local/bin/install-plugins.sh email-ext
RUN /usr/local/bin/install-plugins.sh mailer
RUN /usr/local/bin/install-plugins.sh slack

# Artifacts
RUN /usr/local/bin/install-plugins.sh htmlpublisher

# UI
RUN /usr/local/bin/install-plugins.sh greenballs
RUN /usr/local/bin/install-plugins.sh simple-theme-plugin

# Scaling
RUN /usr/local/bin/install-plugins.sh kubernetes

# install Maven
USER root
RUN apt-get update && apt-get install -y maven
USER jenkins
EOF
```

At this point, we have an active (local) MiniKube Kubernetes cluster and a Jenkins Dockerfile. However, because MiniKube is running the K8s cluster within a local VM it doesn't have access to the built Docker image (if we were to build it outside of the VM).

So we need to build the Docker image from within the VM.

```shell
$ eval $(minikube docker-env)

# -t repo:tag
$ docker build -t ds/jenkins:2.143 .

# verify the availablility of the image
$ docker images
```

```
REPOSITORY                                 TAG                 IMAGE ID            CREATED             SIZE
ds/jenkins                                 2.143                 8967513a875e        17 minutes ago      889MB
```

Now that we have a K8s cluster and a built Docker image, we need to create a K8s deployment configuration. This will tell K8s how to deploy the application within the cluster.

```shell
$ cat > jenkins-deployment.yml <<EOF
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
        - name: jenkins
          image: ds/jenkins:2.143
          env:
            - name: JAVA_OPTS
              value: -Djenkins.install.runSetupWizard=false
          ports:
            - name: http-port
              containerPort: 8080
            - name: jnlp-port
              containerPort: 50000
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-home
          emptyDir: {}
EOF
```

Now to install the created deployment.

```shell
$ kubectl apply -f jenkins-deployment.yml

# Look at the Pod details
$ kubectl describe pod
```

At this point, we only have a K8s Deployment. However, a Service is required to interact with a variable IP Pod (because if the Pod goes down, K8s may bring it back up with a different internal IP).

```shell
$ cat > jenkins-service.yml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: jenkins
EOF
```

Now to add the service.

```shell
$ kubectl create -f jenkins-service.yml
```

Kubernetes, and MiniKube, come with helpful administrative functionality and monitoring which can be accessed via `minikube dashboard`.

So now, let' see if we can go to the Jenkins http dashboard.
{% raw %}
```shell
# Find the ip:port of the Jenkins Service
$ echo $(minikube ip):$(kubectl get services/jenkins -o go-template='{{(index .spec.ports 0).nodePort}}')
```
{% endraw %}

Note: This was just the Jenkins Master. We still need to configure its Slave executors.

## Jenkins Slave Configuration

The blog post I've been configures these executors manually, using Jenkins' console.

I will note the steps here, but hope to find an automated process later.

The Kubernetes Jenkins plugin will require the Jenkins master URL and the internal cluster URL of the Jenkins Pod.

```shell
# Internal URL of Jenkins Pod
$ kubectl cluster-info | grep master
Kubernetes master is running at https://192.168.99.100:8443

# Jenkins master URL
# Get pod name
$ kubectl get pods | grep jenkins
jenkins-6cbd7855db-xpgxh   1/1     Running   0          11m

$ kubectl describe pod jenkins-6cbd7855db-xpgxh | grep IP:
IP:             172.17.0.6
```

In order for Jenkin's Kubernetes cloud executor plugin to connect to the Kubernetes Pod URL, you must bind service account `system:serviceaccount:default:default` (which is the default account bound to a Pod) with role `cluster-admin`. (Note that the yml below will bind the `cluster-admin` role to ALL Pods with `serviceaccount:default:default`). (Adapted from [Fabric GH Issue](https://github.com/fabric8io/fabric8/issues/6840#issuecomment-307560275)).

```shell
$ cat > jenkins-rbac.yml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-rbac
subjects:
  - kind: ServiceAccount
    name: default
    namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```

```shell
$ kubectl apply -f jenkins-rbac.yml

# If you later want to unbind
$ kubectl delete -f jenkins-rbac.yml
```

Navigate to Jenkins Dashboard. Managage Jenkins -> Configure System -> Cloud -> Kubernetes.
```
Name: "kubernetes"
Kubernetes URL: "https://192.168.99.100:8443"  # Internal Jenkins Pod URL
Jenkins URL: "http://172.17.0.6:8080"  # Jenkins master URL - be sure it's http
```

Images -> Kubernetes Pod Template.
```
Name: "jenkins-slave"
Labels: "jenkins-slave"
Containers:
  Container Template
  Name: "jenkins-slave"
  Docker image: "jenkinsci/jnlp-slave"  # using default Jenkins slave image
```

Jenkins -> New Item -> Freestyle project.
```
Name: "first_jenkins_job"
Restrict where this project can run: True
  Label Expression: "jenkins-slave"
Build
  Execute shell
    Command: "sleep 30"
```

Jenkins -> New Item -> copy from "first_jenkins_job"
```
Name: "second_jenkins_job"
```

Build both jobs and watch them sit in the Build Queue while the K8s slave start, then their executors (jenkins-slave-{id}) will show the job build status.

The creation of these slave executors will show in the K8s Dashboard under Pods.

So there is it! We now have a Kubernetes cluster with a master Jenkins Pod which can spawn Jenkins slave executor Pods.


# Future Improvements:
* Jenkins' Dockerfile
    * lock down permissions (remove dashboard warnings)
    * add [Pipeline](https://plugins.jenkins.io/workflow-aggregator) plugin
* Jenkins jobs
    * programmatically configure Pipeline jobs on build server
* Jenkins Configuration
    * define Kubernetes cloud plugin via code
* Kubernetes
    * configure [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) for Jenkins master within deployment yml (if possible)
