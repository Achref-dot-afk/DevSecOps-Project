FROM jenkins/jenkins:lts-jdk11


USER root


RUN apt-get update && \
    curl -fsSL https://get.docker.com | sh


RUN apt-get install -y sudo wget


RUN wget https://github.com/aquasecurity/trivy/releases/download/v0.18.3/trivy_0.18.3_Linux-64bit.deb && \
    dpkg -i trivy_0.18.3_Linux-64bit.deb && \
    rm trivy_0.18.3_Linux-64bit.deb

RUN usermod -aG docker jenkins && \
    gpasswd -a jenkins root

RUN chown -R jenkins:jenkins /var/jenkins_home

USER jenkins

