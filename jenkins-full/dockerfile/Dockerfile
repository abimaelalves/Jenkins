FROM jenkins/jenkins:lts-jdk11
USER root 
WORKDIR /app
RUN apt-get update -y && apt-get install -y curl wget tar vim software-properties-common -y  &&\
     # INSTALL DOCKER
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add - && \
    add-apt-repository \
      "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
      $(lsb_release -cs) \
      stable" &&\
    apt-get update && \
    apt-get -y --no-install-recommends install docker-ce && \
    apt-get clean && \
    usermod -aG docker jenkins && \
    rm -rf /var/lib/apt/lists/* && \     
# JAVA - Instalacao do java na versao 8, pois a versao 11 contem problema de compatibilidade com o cluster criado via kind
    wget --load-cookies /tmp/cookies.txt "https://docs.google.com/uc?export=download&confirm=$(wget --quiet --save-cookies /tmp/cookies.txt --keep-session-cookies --no-check-certificate 'https://docs.google.com/uc?export=download&id=1b1hSZZspy8SFbQ3lZXwB0YoCNN3pxk9j' -O- | sed -rn 's/.*confirm=([0-9A-Za-z_]+).*/\1\n/p')&id=1b1hSZZspy8SFbQ3lZXwB0YoCNN3pxk9j" -O jdk-8u191-linux-x64.tar.gz && rm -rf /tmp/cookies.txt &&\
    rm -rf /opt/java && \
    mkdir /opt/java &&\
    cp /app/jdk-8u191-linux-x64.tar.gz /opt/java &&\
    cd /opt/java && tar xzf jdk-8u191-linux-x64.tar.gz &&\
    update-alternatives --install /usr/bin/java java /opt/java/jdk1.8.0_191/bin/java 1 &&\
    export JAVA_HOME=/opt/jdk1.8.0_191 &&\
    rm -f /app/jdk-8u191-linux-x64.tar.gz && rm -f /opt/java/jdk-8u191-linux-x64.tar.gz
USER jenkins
