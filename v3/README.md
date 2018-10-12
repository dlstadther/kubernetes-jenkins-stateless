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
docker build -t ds/jenkins:2.145 .

kubectl apply -f jenkins.yml

# VIEW
# get http address for jenkins
echo $(minikube ip):$(kubectl get services/jenkins -o jsonpath='{.spec.ports[0].nodePort}')

# DELETE
kubectl delete -f jenkins.yml

docker rmi sample/jenkins:2.143

# STOP
minikube stop
```

## Summary

Now that I have learned how to create a Jenkins Kubernetes cluster by following a series of manual steps and then iterating upon them to reduce the project artifacts, I wanted to go a step further and remove the manual setups of configuring Jenkins - enter Jenkins Configuration as Code (JCasC).

To refine the goal of this project iteration, I settled on the target of a fully configured Jenkins cluster with Kubernetes containerized executors with a single Pipeline job. For extra credit, I also wanted to reach an end result where Jenkins did not complain about any security warnings at the time of instance launch.

I will talk about the changes in two section: 1) Jenkins Configuration as Code and 2) Kubernetes.

### Jenkins Configuration as Code (Intro)

Still in its early stages (v1.1), the Jenkins Configuration as Code plugin enables you to specify your Jenkins configuration in a YAML and reference that file location. In practice, it requires 3 things - 1) the JCasC plugin, 2) a YAML configuration file on your server, and 3) the `CASC_JENKINS_CONFIG` environment variable set to the fully qualified path name of your YAML configuration file.

Manually utilizing JCasC (as a proof of concept) was pretty easy. Install the JCasc plugin, ssh into the Jenkins server as the Jenkins service user, create a basic JCasC YAML, and tell the Jenkins plugin to load from a file path location.

```shell
$ cat > /var/jenkins_home/jcasc.yml <<EOF
jenkins:
  systemMessage: "Welcome to Jenkins - configured with code!"
  labelString: "jenkins-master"
  mode: EXCLUSIVE
  numExecutors: 3
tool:
  git:
    installations:
    - home: "git"
      name: "Default"
EOF
```

### Kubernetes Facilitation

Now that we know what we want to accomplish (adding a plugin, generating a remote file, and setting an environment variable), we just have to identify how to utilize Kubernetes to reach this goal.

Adding the plugin and setting an environment variable are definitely the easy given that our previous iterations have already included some of both.

Include the following in `plugins.txt`. Then rebuild Docker image.
```txt
configuration-as-code:1.1
configuration-as-code-support:1.1
jdk-tool:1.1
```

Append this `env` to your preexisting spec->template->spec->containers K8s Deployment definition.
```yml
env:
  - name: CASC_JENKINS_CONFIG
    value: /path/to/jcasc.yml
```

Lastly, we need a way to create a file and ensure it gets deployed with the Jenkins instance. To accomplish this, we're going to use a Kubernetes `ConfigMap`. The purposes of ConfigMaps is to decouple configuration from image content such to ensure an image can be portable across multiple applications. To deploy the contents of a ConfigMap to a Deployment, we only need to add a `volumeMount` and a `volume` which references our `ConfigMap`.

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jcasc
data:
  jcasc.yml: |
    jenkins:
      systemMessage: "Welcome to Jenkins - configured with code!"
      labelString: "jenkins-master"
      mode: EXCLUSIVE
      numExecutors: 3
    tool:
      git:
        installations:
        - home: "git"
          name: "Default"

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
...
spec:
  ...
  template:
    ...
    spec:
      ...
      containers:
        volumeMounts:
          - name: jcasc
            mountPath: /var/jenkins_config
      volumes:
        - name: jcasc
          configMap:
            name: jcasc
```

Now it works just like we expect, right?! WRONG! Jenkins will come online, but without any of our configuration changes.

### The Straw that Broke the Camel's Back

After many hours, the source of the problem (or at least as far as I've uncovered, shy of the proper fix) revolves around how ConfigMaps are stored in volumeMounts AND how JCasC reads its configuration.

Kubernetes supports the update of ConfigMap on a running Pod and they are able to do so through how the volume data is stored. In our case, it may look something like this:

```shell
$ MY_PATH = /var/jenkins_config/
$ ls -alh $MY_PATH
total 12K
drwxrwxrwx 3 root root 4.0K Oct  11 00:35 .
drwxr-xr-x 1 root root 4.0K Oct  11 00:38 ..
drwxr-xr-x 2 root root 4.0K Oct  11 00:35 ..2018_10_11_00_35_45.759292262
lrwxrwxrwx 1 root root   31 Oct  11 00:35 ..data -> ..2018_10_11_00_35_45.759292262
lrwxrwxrwx 1 root root   19 Oct  11 00:35 jcasc.yml -> ..data/jcasc.yml
```
Basically, our ConfigMap data gets placed in a hidden folder based on the current timestamp (`..2018_10_11_00_35_45.759292262`) with a symbolic link (`..data`) to it which is timestamp-agnostic. Finally, a symbolic link file with the name of our ConfigMap's data redirects through two symbolic links to the source data.

Then I replace all applicable parts of the config (including the ConfigMap)
```shell
$ kubectl replace -f jenkins.yml
```

So if we go back to the volumeMount location, we see that the contents changed slightly.
```shell
$ MY_PATH = /var/jenkins_config/
$ ls -alh $MY_PATH
total 12K
drwxrwxrwx 3 root root 4.0K Oct 12 01:23 .
drwxr-xr-x 1 root root 4.0K Oct 11 00:38 ..
drwxr-xr-x 2 root root 4.0K Oct 12 01:23 ..2018_10_12_01_23_47.044256661
lrwxrwxrwx 1 root root   31 Oct 12 01:23 ..data -> ..2018_10_12_01_23_47.044256661
lrwxrwxrwx 1 root root   32 Oct 11 01:28 jcasc.yml -> ..data/jcasc.yml
```
The timestamp directory was updated with a fresh `..data` symbolic link.

At this point, you can confirm that the filesystem understands how to evaluate `/var/jenkins_config/jcasc.yml`.

```shell
$ cat /var/jenkins_config/jcasc.yml
jenkins:
  systemMessage: "Welcome to Jenkins - configured with code!"
  labelString: "jenkins-master"
  mode: EXCLUSIVE
  numExecutors: 3
tool:
  git:
    installations:
    - home: "git"
      name: "Default"
```

BUT JCasC can't read and interpret the data contents when referencing `/var/jenkins_config/jcasc.yml`.

The final mystery resides somewhere in JCasC or Java because the reference removal of one of the symbolic links resolves the issue.

```yml
env:
  - name: CASC_JENKINS_CONFIG
    # https://github.com/jenkinsci/configuration-as-code-plugin/issues/425
    value: /var/jenkins_config/..data/jcasc.yml
```

We now have a functional Jenkins instance, configured through code. However, I'm not very happy with this patch solution and hope to spend more time (later) to investigate specifically why JCasC can't reference the double symbolic linked filename to resolve the server's configuration YAML.

But alas, I needed to continue towards my goal of this iteration.

### Extra Credit - Security Management
Whenever Jenkins is initially booted, you're almost always presenting with a red box with a number along the browser banner to denote the number of warnings you have at that time. These can range from "there's a new Jenkins version" to "you have security vulnerabilities we recommend you patch".

Not only do I want to reach the goal of instantiating a stateless Jenkins instance, but I want it to be "production-ready", clean, and ready to go! These warnings have to go away!

The process of removal, ultimately just involved an interative approach by understanding the error, finding how to make it go away, and implementing the solution into the JCasc configuration.

```yml
$ cat /var/jenkins_config/jcasc.yml
jenkins:
  systemMessage: "Welcome to Jenkins - configured with code!"
  labelString: "jenkins-master"
  mode: EXCLUSIVE
  numExecutors: 3
  crumbIssuer: "standard"
  remotingSecurity:
    enabled: true
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: admin
          password: root
        - id: doge
          password: wow
security:
  remotingCLI:
    enabled: false
unclassified:
  location:
    url: http://192.168.99.100:30123/
tool:
  git:
    installations:
    - home: "git"
      name: "Default"
```

The final security patch I implemented in this revision was to the applied Kubernetes `Role`/`RoleBinding` to allow Jenkins to spawn containerized executor Pods. I won't go over that here, but you can its changes in the `Role` sections of the Kubernetes definition.

### Coming Next

The next iteration of this project will be a refactor upon this version to add more complexity to the JCasC setup.
