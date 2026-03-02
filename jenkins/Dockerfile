FROM jenkins/jenkins:lts-jdk21

USER root

# Installer Docker CLI et kubectl
RUN apt-get update && apt-get install -y docker.io curl && \
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && \
    install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl && \
    rm kubectl

# Installer Helm
RUN curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

RUN groupdel docker || true && \
    groupadd -g 112 docker && \
    usermod -aG docker jenkins

USER jenkins