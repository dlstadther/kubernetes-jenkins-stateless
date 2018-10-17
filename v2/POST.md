Now that I successfully followed a blog post and created a Jenkins slave-executor within Kubernetes, the next goal is to refactor the multiple K8s configurations into a single YML and make small Docker and K8s changes.

## Create Local Dir
```shell
# Navigate to local dir
$ mkdir ~/tutorial/k8s/jenkins/v2
$ cd ~/tutorial/k8s/jenkins/v2
$ minikube start
```

## Create plugins.txt
```shell
# List plugins for Jenkins image
$ cat > plugins.txt <<EOF
ssh-slaves

email-ext
mailer
slack

htmlpublisher

greenballs
simple-theme-plugin

kubernetes

workflow-aggregator

EOF
```

## Define and Build Jenkins Dockerfile with plugins.txt
```shell
# Define Jenkins image
$ cat > Dockerfile <<EOF
from jenkins/jenkins:2.143

COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

USER root
RUN apt-get update && apt-get install -y maven
USER jenkins

EOF
```

```shell
# Build Jenkins image for Minikube
$ eval $(minikube docker-env)
$ docker build -t ds/jenkins-v2:2.143 .
```

## Refactor K8s Config and Add ServiceAccount Name to Jenkins Pod
```shell
# Define K8s
$ cat > jenkins.yml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
spec:
  replicas: 1
  template:
    metadata:
      name: jenkins
      labels:
        name: jenkins
        app: jenkins
    spec:
      serviceAccountName: jenkins
      containers:
        - name: jenkins
          image: ds/jenkins-v2:2.143
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

---
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

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```

```shell
# Apply K8s definitions
$ kubectl apply -f jenkins.yml

# Get Jenkins URL
$ echo $(minikube ip):$(kubectl get services/jenkins -o go-template='{{(index .spec.ports 0).nodePort}}')
```


## Manually configure K8s Jenkins executors

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

Manage Jenkins -> Configure System -> Usage (only build jobs with label expressions matching this node).

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


## Remove Jenkins Pod and Docker Image
```shell
$ kubectl delete -f jenkins.yml
$ docker images
REPOSITORY                                 TAG                 IMAGE ID            CREATED             SIZE
ds/jenkins-v2                              2.143               d90911d56d00        31 minutes ago      775MB

$ docker rmi d90911d56d00
```
