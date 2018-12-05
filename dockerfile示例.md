##  制作tomcat镜像
```
FROM centos
MAINTAINER nobody "xx@qq.com"
RUN mkdir -p /opt/jdk/
RUN mkdir -p /opt/tomcat/
ADD jdk1.8.0_79 /opt/jdk/
ADD tomcat  /opt/tomcat/
ENV CATALINA_HOME /opt/tomcat
ENV JAVA_HOME /opt/jdk
EXPOSE 8080
ENV PATH $PATH:$JAVA_HOME/bin
CMD ["/opt/tomcat/bin/catalina.sh","run"]
```
##  制作tomcat镜像2
```
FROM ubuntu
MAINTAINER xrzp@qq.com

ENV VERSION=8.5.31

ENV JAVA_HOME /usr/local/jdk
COPY apache-tomcat-8.5.31.tar.gz .
RUN apt-get update && \
    apt-get install wget curl unzip iproute2 net-tools -y && \
    apt-get clean all && \
    tar zxf apache-tomcat-${VERSION}.tar.gz && \
    mv apache-tomcat-${VERSION} /usr/local/tomcat && \
    rm -rf apache-tomcat-${VERSION}.tar.gz /usr/local/tomcat/webapps/* && \
    sed -i '1a JAVA_OPTS="-Djava.security.egd=file:/dev/./urandom"' /usr/local/tomcat/bin/catalina.sh && \
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

WORKDIR /usr/local/tomcat

EXPOSE 8080
CMD ["./bin/catalina.sh", "run"]
```
docker build -t 192.168.3.27:8888/ops/tomcat85
## 利用上面制作好的tomcat镜像部署应用
```
FROM 192.168.3.27:8888/ops/tomcat85
ADD target/solo.war /tmp
RUN unzip -q /tmp/solo.war -d /usr/local/tomcat/webapps/ROOT

COPY deploy/docker-entrypoint.sh /usr/bin/docker-entrypoint.sh
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 8080
CMD ["./bin/catalina.sh", "run"]

```

## 制作安装了软件的 基础镜像
```
FROM debian:stretch
MAINTAINER suredandan xrzp@qq.com

ENV TIMEZONE=Asia/Shanghai \
    LANG=zh_CN.UTF-8

RUN echo "${TIMEZONE}" > /etc/timezone \
    && echo "$LANG UTF-8" > /etc/locale.gen \
    && apt-get update -q \
    && ln -sf /usr/share/zoneinfo/${TIMEZONE} /etc/localtime \
    && mkdir -p /home/jenkins/.jenkins \
    && mkdir -p /home/jenkins/agent \
    && mkdir -p /usr/share/jenkins 

# COPY chhostname.sh /usr/local/bin/chhostname.sh

# java/locale/DinD/svn/jnlp
RUN  DEBIAN_FRONTEND=noninteractive apt-get install -yq vim wget curl apt-utils dialog locales apt-transport-https build-essential bzip2 ca-certificates sudo jq unzip zip gnupg2 software-properties-common \ 
     && update-locale LANG=$LANG \
     && locale-gen $LANG \
     && DEBIAN_FRONTEND=noninteractive dpkg-reconfigure locales \
     && curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/debian/gpg | sudo apt-key add - \
     && add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/debian $(lsb_release -cs) stable" \
     && apt-get update -y \
     && apt-get install -y docker-ce \
     && apt-get install -y subversion \
     && groupadd -g 10000 jenkins \
     && useradd -c "Jenkins user" -d $HOME -u 10000 -g 10000 -m jenkins \
     && usermod -a -G docker jenkins \
     && sed -i '/^root/a\jenkins    ALL=(ALL:ALL) NOPASSWD:ALL' /etc/sudoers

USER root

WORKDIR /home/jenkins
```
`docker build -t 192.168.3.27:8888/ops/os2 .`

## 从上个镜像继续制作jnlp镜像
```
FROM 192.168.3.27:8888/ops/os2
MAINTAINER suredandan xrzp@qq.com

RUN mkdir -p /usr/local/maven \
    && mkdir -p /usr/local/jdk \    
    && mkdir -p /root/.kube

COPY jdk /usr/local/jdk
COPY maven /usr/local/maven
COPY kubectl /usr/local/bin/kubectl 
COPY jenkins-slave /usr/local/bin/jenkins-slave 
COPY slave.jar /usr/share/jenkins
COPY config /root/.kube/

ENV JAVA_HOME=/usr/local/jdk \
    MAVEN_HOME=/usr/local/maven \
    PATH=/usr/local/jdk/bin:/usr/local/maven/bin:$PATH 
	
ENTRYPOINT ["jenkins-slave"]
```

## 将应用 部署到基础镜像
```
docker pull java
docker tag java 192.168.3.27:8888/ops/java
```
利用上面的java 镜像 部署我们的java应用
```
FROM 192.168.3.27:8888/ops/java
MAINTAINER suredandan xrzp@qq.com

ADD target/devPortal-web-1.0.0-SNAPSHOT.jar /tmp

EXPOSE 9527
ENTRYPOINT java -jar /tmp/devPortal-web-1.0.0-SNAPSHOT.jar
```
