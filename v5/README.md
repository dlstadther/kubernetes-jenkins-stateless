# V5: Jenkins/K8s
v4 + multi-job and syntax

## Execution
```shell
# START VM
minikube start
minikube dashboard

# NAVIGATE
# cd to dir with the supporting k8s/docker files
cd ~/code/k8s-jenkins/v5/

# BUILD
eval $(minikube docker-env)
docker build -t ds/jenkins-v5:2.147 .

kubectl apply -f jenkins.yml

# VIEW
# get http address for jenkins
echo $(minikube ip):$(kubectl get services/jenkins-ui -o jsonpath='{.spec.ports[0].nodePort}' --namespace jenkins)

# DELETE
kubectl delete -f jenkins.yml

docker rmi ds/jenkins-v5:2.147

# STOP
minikube stop
```

## Summary
The current iteration of this project will be a refactor upon the previous version to add multiple build jobs with multiple syntaxes.
