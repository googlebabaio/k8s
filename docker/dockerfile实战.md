<!-- toc -->

# 镜像的制作分为三类
- 基础镜像
- 运行环境镜像
- app镜像


# 制作基础镜像
## 制作一个安装了扩展操作系统的基础镜像
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

推送到私有镜像仓库中:
`docker build -t 192.168.3.27:8888/ops/os2 .`

### 基于这个基础镜像`os2`制作jenkins的slave镜像

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

# 制作运行环境镜像

## 直接拖取已有的java运行环境镜像
```
docker pull java
docker tag java 192.168.3.27:8888/ops/java
```

##  制作tomcat镜像
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
将镜像推送到私有镜像仓库
`docker build -t 192.168.3.27:8888/ops/tomcat85`

## 直接从基础镜像制作nginx镜像
```
FROM centos:7
MAINTAINER xrzp@qq.com
RUN yum install -y gcc gcc-c++ make \
    openssl-devel pcre-devel gd-devel libxslt-devel \
    iproute net-tools telnet wget curl && \
    yum clean all && \
    rm -rf /var/cache/yum/*
RUN wget http://nginx.org/download/nginx-1.15.6.tar.gz && \
    tar zxf nginx-1.15.6.tar.gz && \
    cd nginx-1.15.6 && \
    ./configure --prefix=/usr/local/nginx \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-http_realip_module \
    --with-http_image_filter_module \
    --with-http_gunzip_module \
    --with-http_gzip_static_module \
    --with-http_secure_link_module \
    --with-http_stub_status_module \
    --with-stream \
    --with-stream_ssl_module && \
    make -j 4 && make install && \
    mkdir -p /usr/local/nginx/conf/vhost && \
    rm -rf /usr/local/nginx/html/* && \
    echo "ok" >> /usr/local/nginx/html/status.html && \
    cd / && rm -rf nginx-1.15.6*
ENV PATH $PATH:/usr/local/nginx/sbin
WORKDIR /usr/local/nginx
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
将镜像推送到私有镜像仓库
`docker build -t 192.168.3.27:8888/ops/nginx`

## 制作php镜像
```
FROM centos:7
MAINTAINER xrzp@qq.com
RUN yum install epel-release -y && \
    yum install -y gcc gcc-c++ make gd-devel libxml2-devel \
    libcurl-devel libjpeg-devel libpng-devel openssl-devel \
    libmcrypt-devel libxslt-devel libtidy-devel autoconf \
    iproute net-tools telnet wget curl && \
    yum clean all && \
    rm -rf /var/cache/yum/*

RUN wget http://docs.php.net/distributions/php-5.6.36.tar.gz && \
    tar zxf php-5.6.36.tar.gz && \
    cd php-5.6.36 && \
    ./configure --prefix=/usr/local/php \
    --with-config-file-path=/usr/local/php/etc \
    --with-config-file-scan-dir=/usr/local/php/etc/php.d \
    --enable-fpm --enable-opcache --enable-static=no \
    --with-mysql --with-mysqli --with-pdo-mysql \
    --enable-phar --with-pear --enable-session \
    --enable-sysvshm --with-tidy --with-openssl \
    --with-zlib --with-curl --with-gd --enable-bcmath \
    --with-jpeg-dir --with-png-dir --with-freetype-dir \
    --with-iconv --enable-posix --enable-zip \
    --enable-mbstring --with-mhash --with-mcrypt --enable-hash \
    --enable-xml --enable-libxml --enable-debug=no && \
    make -j 4 && make install && \
    cp php.ini-production /usr/local/php/etc/php.ini && \
    cp sapi/fpm/php-fpm.conf /usr/local/php/etc/php-fpm.conf && \
    sed -i "90a \daemonize = no" /usr/local/php/etc/php-fpm.conf && \
    mkdir /usr/local/php/log && \
    cd / && rm -rf php*

ENV PATH $PATH:/usr/local/php/sbin
WORKDIR /usr/local/php
EXPOSE 9000
CMD ["php-fpm"]
```

# 制作app镜像
## 利用上面制作好的`tomcat85`镜像部署应用
```
FROM 192.168.3.27:8888/ops/tomcat85
ADD target/solo.war /tmp
RUN unzip -q /tmp/solo.war -d /usr/local/tomcat/webapps/ROOT

COPY deploy/docker-entrypoint.sh /usr/bin/docker-entrypoint.sh
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 8080
CMD ["./bin/catalina.sh", "run"]
```

## 将应用部署到`java`运行环境的镜像
利用上面的java 镜像 部署我们的java应用
```
FROM 192.168.3.27:8888/ops/java
MAINTAINER suredandan xrzp@qq.com

ADD target/devPortal-web-1.0.0-SNAPSHOT.jar /tmp

EXPOSE 9527
ENTRYPOINT java -jar /tmp/devPortal-web-1.0.0-SNAPSHOT.jar
```
