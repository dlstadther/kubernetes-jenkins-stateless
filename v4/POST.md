The goal of the current iteration of this project is to expand on the Jenkins Configuration as Code plugin usage and begin "hardening" the stateless Jenkins instance.

Specifically, I updated my JCasC to setup Kubernetes slave executors with a Pipeline build job, include some additional security configuration, and update the Kubernetes recipe to use a non-default namespace. I won't directly address the additional security configuration below - it was as straight forward as installing the appropriate plugin(s) and updating the JCasC as one would believe.

## JCasC - jobs
To start, I wanted to add a single Pipeline build job to the JCasC framework. Since all the examples I've seen online used the Scripted syntax, I did the same.

```yml
jobs:
  - script: >
      pipelineJob('CasC - DSL Scripted') {
        definition {
          cps {
            script('''
      properties([parameters([booleanParam(defaultValue: false, description: '', name: 'isRelease')])])

      node() {
        stage("checkout") {
          echo "Checking out..."
        }

        stage("build") {
          echo "Building..."
          sleep 2
        }

        stage("release") {
          if(!params?.isRelease) {
            echo "Release build is not enabled"
          } else {
            echo "Release build enabled. Running release."
            sleep 15
          }
        }
      }
            ''')
            sandbox()
          }
        }
      }
```

If you only add this to your `jcasc.yml` and apply it to your Jenkins instance, you'll get an error related to `jobs` being an unknown element of JCasC. This is resolved by adding the plugin `job-dsl`.

## JCasC - kubernetes executors
Earlier versions of this project included a manual setup of Jenkins' Kubernetes executors. My desire to automate that setup is ultimately what led me up through this project revision. However, I didn't realize how complicated it was going to get to reach the goal of automating this portion of Jenkins' configuration.

But wait, can't this just be included in the JCasC config and be good to go?! Well, yes. Sort of.

The kicker in configuring the Kubernetes executors is their required knowledge of the Kubernetes master URL and Jenkins' master URL. Neither of these URLs are guaranteed to be static - nor would you want to force them to be either. My initial attempt at solving this was for the Jenkins `Deployment` to include a few environment variables relating to the needed IPs and Ports - then just referencing those ENV names in the JCasC.yml. However, I quickly found out that Jenkins' configuration reads `${MY_VAR}` as raw strings and doesn't convert them into their system environment variable values. Back to the drawing board.

Unsurprisingly, the Kubernetes community had already solved for this problem (well, not my exact problem but the general idea)! Enter cluster DNS.

When assigning a service to an selector and namespace, Kubernetes DNS assigns a reference to that service using the format `metadata-name.namespace` (which can also be referenced by `metadata-name.namespace.svc.cluster.local`). There is mostly definitely more to it than that, but for my current need that is as far as I followed the rabbit hole.

A second issue arose surrounding the ability for the master node to receive appropriate response from the slaves upon startup. This was resolved by providing the `jenkinsTunnel` argument with a DNS reference to a `ClusterIP` Kubernetes `Service` pointed at the port `50000` designated for jnlp-slaves.

```yml
jenkins:
  ...
  clouds:
    - kubernetes:
        name: kubernetes
        namespace: jenkins
        serverUrl: https://kubernetes.default
        jenkinsUrl: http://jenkins-ui.jenkins:8080  # points to NodePort Service for Jenkins port 8080
        jenkinsTunnel: jenkins-discovery.jenkins:50000  # points to ClusterIP Service for Jenkins port 50000
        ...
```

## Kubernetes - namespace
As seen from the example configuration above, there are two namespaces being referenced (in the A-Records) - `default` and `jenkins`. Up to this point, I had only been using the `default` namespace - which is used if one is not specified. I created a `Namespace` definition and assigned all Jenkins-related artifacts to that namespace.

```yml
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
  labels:
    name: jenkins
```

Kubernetes artifacts can be assigned a namespace in their YML definition by adding `namespace: jenkins` within their `metadata`.
