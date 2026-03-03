FROM jenkins/jenkins:lts-jdk21

USER root

# Installer Docker CLI et kubectl
RUN apt-get update && apt-get install -y docker.io curl && \
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && \
    install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl && \
    rm kubectl

# Installer Helm — téléchargement direct du binaire
RUN curl -LO https://get.helm.sh/helm-v3.17.0-linux-amd64.tar.gz && \
    tar -zxvf helm-v3.17.0-linux-amd64.tar.gz && \
    mv linux-amd64/helm /usr/local/bin/helm && \
    rm -rf helm-v3.17.0-linux-amd64.tar.gz linux-amd64

RUN groupdel docker || true && \
    groupadd -g 112 docker && \
    usermod -aG docker jenkins

USER jenkins