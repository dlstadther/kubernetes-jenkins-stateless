from jenkins/jenkins:2.147
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

# install system dependencies
USER root
# Python2 and Pip
RUN apt-get update && apt-get install -y python python-pip python-virtualenv

USER jenkins
