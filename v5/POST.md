A build server for a single job is an excessive use of a build server. It has come time to add more jobs, using different syntaxes and dependencies. The goal of this revision is simple - add more job diversity.

Moving from an educational project into something practical is challenging. It was this revision that was supposed to bridge that gap.
The original intent of this version was to add multiple job definitions into the CasC configMap, specifically one with actual application.

I say "original intent" because a compromise was required to denote this revision as "complete". We'll come to this later in the post.

## Multi-Job - Proof
To prove that multiple jobs could be configured with CasC and job-dsl, I kept it simple and just added a second copy of the original job - one that had been proven to be syntacically correct.

```yml
jobs:
  - script: >
      pipelineJob('CasC - DSL Scripted 0') {
        definition {
          cps {
            script('''
      properties([parameters([booleanParam(defaultValue: false, description: '', name: 'isRelease')])])

      node('slave-default') {
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
  - script: >
      pipelineJob('CasC - DSL Scripted 1') {
        definition {
          cps {
            script('''
      properties([parameters([booleanParam(defaultValue: false, description: '', name: 'isRelease')])])

      node('slave-default') {
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

As hoped, this worked as expected. Both ran correctly, without problem.

> It's worth noting here that I did add the node agent declarations with `slave-default` - the new label for my Kubernetes slave executors.

## Multi-Job, Multi-Syntax
Up to this point, I have only used the Pipeline Scripted syntax for CasC job configuration. Honestly, this is only due to availability of example code - everyone was using this syntax. However, in practice I've only ever used Pipeline's Declarative syntax. So can job-dsl and CasC interpret both alongside each other?

```yml
jobs:
  - script: >
      pipelineJob('CasC - DSL Scripted 0') {
        definition {
          cps {
            script('''
      properties([parameters([booleanParam(defaultValue: false, description: '', name: 'isRelease')])])

      node('slave-default') {
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
  - script: >
      pipelineJob('CasC - DSL Declarative 0') {
        definition {
          cps {
            script('''
      pipeline {
        agent {
          label 'slave-default'
        }
        options {
          timeout(time: 30, unit: 'MINUTES')
          timestamps()
        }
        stages {
          stage ('build') {
            steps {
              sleep 2
            }
          }
          stage ('deploy:prod') {
            steps {
              sleep 15
            }
          }
        }
      }
            ''')
            sandbox()
          }
        }
      }
```

Again, we have success! So long as each `script` record abides by the [Job DSL syntax](https://jenkinsci.github.io/job-dsl-plugin/#), CasC appears to configure successfully.

## Multi-Job - Multi-Sample

```yml
jobs:
  - script: >
      pipelineJob('CasC - DSL Scripted 0') {
        definition {
          cps {
            script('''
      properties([parameters([booleanParam(defaultValue: false, description: '', name: 'isRelease')])])

      node('slave-default') {
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
  - script: >
      pipelineJob('CasC - DSL Scripted 1') {
        definition {
          cps {
            script('''
      node('slave-default') {
        stage("checkout") {
          echo "Checking out..."
        }

        stage("build") {
          echo "Building..."
          sleep 2
        }

        stage("release") {
          echo "Release build enabled. Running release."
          sleep 15
        }
      }
            ''')
            sandbox()
          }
        }
      }
  - script: >
      pipelineJob('CasC - DSL Declarative 0') {
        definition {
          cps {
            script('''
      pipeline {
        agent {
          label 'slave-default'
        }
        options {
          timeout(time: 30, unit: 'MINUTES')
          timestamps()
        }
        stages {
          stage ('build') {
            steps {
              sh "sleep 2"
            }
          }
          stage ('deploy:prod') {
            steps {
              sh"sleep 15"
            }
          }
        }
      }
            ''')
            sandbox()
          }
        }
      }
```

Still works!

## Job DSL with Dependencies
It was now time to attempt to implement a new job with practical purposes. Given my responsibility with Spotify's Luigi, I thought to utilize this build server to run some of Luigi's tox tests.

In order to run these tests, I needed Python and virtualenv installed on my build server.

### Master System Dependencies
At this point, neither my Jenkins master image nor Jnlp slaves images have Python nor virtualenv installed. So to begin, I decided to reconfigure my master to have one executor. Then I rebuilt its image to install Python 2 and virtualenv.

```yml
jenkins:
  systemMessage: "V5: Stateless Jenkins w/ JCasC!!!"
  labelString: master
  ...
  mode: EXCLUSIVE
  numExecutors: 1
  ...
```

```Dockerfile
from jenkins/jenkins:2.147
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

# install system dependencies
USER root
RUN apt-get update && apt-get install -y python python-pip python-virtualenv
USER jenkins
```

Then rebuild the Jenkins master image.
```shell
$ docker build -t ds/jenkins-v5:2.147 .
```

Next, I needed to add a simple Luigi job.

```yml
- script: >
    pipelineJob('spotify-luigi') {
      displayName("spotify-luigi")
      description("Build environment for Spotify's Luigi")
      definition {
        cps {
          script('''
    pipeline {
      agent {
        label 'master'
      }
      stages {
        stage ('checkout') {
          steps {
            git url: 'https://github.com/spotify/luigi.git', branch: 'master'
          }
        }
        stage ('prepare venv') {
          steps {
            sh """
              python -m virtualenv ~/venv
              ~/venv/bin/pip install tox
            """
          }
        }
        stage ('test-flake8') {
          steps {
            sh """
              ~/venv/bin/tox -e flake8
            """
          }
        }
      }
    }
          ''')
          sandbox()
        }
      }
    }
```

Alternatively, I could have `pip install tox` at the system level and not dealt with virtualenv. But I've come to use virtualenv for all Python execution environments in order to minimize system-level Python dependencies.

At this point, we're still successful!

### Jenkins Agent System Dependencies
Now that I successfully got Python installed and executed on Jenkins master, I needed to do the same for the Jnlp Slaves. Easy, right? Complete wrong.

> Here is where my "original intent" departed from the actual implementation.

I believed the correct solution here was to build an image based off the jenkinsci/jnlp-slave image with the inclusion of Python, pip, and virtualenv.

```Dockerfile
from jenkins/jnlp-slave:latest

# install system dependencies
USER root
RUN apt-get update && apt-get install -y python python-pip python-virtualenv
USER jenkins
```

```shell
$ docker build -t ds/jnlp-slave-python:latest .
```

However, the use of this image as one of my executor agent images always results in a `python command not found` error when attempting to create a virtualenv.

> Note that I have specifically chosen not to use the Jenkins plugin `shiningpanda`, as I want the ability to control the specific Python (and other dependency) versions installed on my images.

After spending multiple days on this issue, I gave up temporarily in the interest of maintaining momentum on this project. For now, I have left a single master executor where this job will run.

## Conclusion
This project revision was both a success and a failure. I managed to prove that CasC and job-dsl would configure a build server with multiple jobs, in multiple syntaxes. However, I was unable to correctly configure slave agents with the appropriate system dependencies to run Python tests.

The last thing I wanted was to get hung up on an issue where I lost motivation to iterate upon this project. I made the decision to allow this failure to persist so that I could continue to learn more Kubernetes and Jenkins configuration concepts, with the intention of revisiting this error later down the educational road.

Feel free to reach out to me if you believe to know the missing piece here.
