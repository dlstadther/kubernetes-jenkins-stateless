# V3: Jenkins/K8s
v2 + Jenkins Configuration as Code

## Execution
```shell
# START VM
minikube start
minikube dashboard

# NAVIGATE
# cd to dir with the supporting k8s/docker files
cd ~/code/k8s-jenkins/v3/

# BUILD
eval $(minikube docker-env)
docker build -t ds/jenkins-v3:2.146 .

kubectl apply -f jenkins.yml

# VIEW
# get http address for jenkins
echo $(minikube ip):$(kubectl get services/jenkins -o jsonpath='{.spec.ports[0].nodePort}')

# DELETE
kubectl delete -f jenkins.yml

docker rmi ds/jenkins-v3:2.146

# STOP
minikube stop
```
