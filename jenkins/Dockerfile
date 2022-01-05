FROM trion/jenkins-docker-client:latest
# FROM jenkins/jenkins:lts-jdk11 # TODO install docker client to only use the official image

RUN jenkins-plugin-cli --plugins gitea branch-api workflow-multibranch workflow-aggregator docker-plugin docker-workflow configuration-as-code ssh-agent

USER root
RUN groupadd docker && usermod -aG docker jenkins && usermod -aG root jenkins
# USER jenkins
