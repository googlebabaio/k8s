http://blog.51cto.com/andyxu/2296339

示例：

### docker build -t nginx:15.6 .
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


### php
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