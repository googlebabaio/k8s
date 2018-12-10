<!-- toc -->
## build命令
和dockerfile配套使用的指令,根据dockerfile进行进行打包。
```
Usage:  docker build [OPTIONS] PATH | URL | -
OPTIONS：
-t， --tag list  #指定构建的镜像名称和标记名称
-f， --file string #指定Dockerfiile文件路径
```
示例：
1. `docker build .` #不指定镜像名称的话，将会默认以<none>作为镜像名称和标记名称
2. `docker build -t myip:v1 .` #镜像名称为nginx，标记名称为v1
3. `docker build -t myip:v1 -f /path/Dockerfile` /path #指定dockerfile文件路径


> 说明：
在构建镜像时，Docker daemon会首先将Dockerfile所在的目录构建成一个context（上下文），然后通过Dockerfile里的COPY或者ADD语句将context里的文件复制到指定的镜像目录里。
所以需要复制到镜像里的文件，无论是脚本、安装包还是配置文件，都需要跟Dockerfile文件放在同一个目录下。
当执行docker build命令后，返回的第一条信息便是在构建上下文。

## Dockerfile的基本指令
基本指令有十多个，分别是：
* FROM
* MAINTAINER
* ENV
* RUN
* CMD
* EXPOSE
* ARG
* ADD
* COPY
* ENTRYPOINT
* VOLUME
* USER
* WORKDIR
* HEALTHCHECK
* ONBUILD

### FROM
语法：
```
FROM <image>
```
>说明：第一个指令必须是FROM了，其指定一个构建镜像的基础源镜像，如果本地没有就会从公共库中拉取，没有指定镜像的标签会使用默认的latest标签，可以出现多次，如果需要在一个Dockerfile中构建多个镜像。

### MAINTAINER
语法：
```
MAINTAINER <name> <email>
```
>说明：描述镜像的创建者，名称和邮箱

### ENV
用法：
```
ENV <key> <value>   #设置一个环境变量
ENV <key1>=<value1> <key2>=<value2>...   #设置多个环境变量
```
>
说明：设置容器的环境变量，可以让其后面的RUN命令使用，容器运行的时候这个变量也会保留。

### RUN
语法：
```
RUN <command>
RUN ["executable", "param1", "param2" ...]
```

说明：构建镜像时运行的shell命令
例如：`RUN yum install httpd`或者`RUN ["yum", "install", "httpd"]`
Dockerfile中每一个指令都会建立一层，RUN也不例外。多少个RUN就构建了多少层镜像，多个RUN会产生非常臃肿、多层的镜像，不仅仅增加了构建部署的时间，还容易出错。我们在写Dockerfile的时候，要尽量避免使用多个RUN，尽量将需要执行的命令都写在一个RUN里。多条命令可使用\来换行，换行的命令前面加上&&，最后还需要将不需要的文件及目录删除，比如安装包、临时目录等，减少容器的大小。
注：Union FS是有最大层数限制的，比如AUFS，曾经是最大不得超过42层，现在是不得超过127层。

### CMD
用法：
```
CMD <命令>
CMD ["可执行文件", "参数1", "参数2"...]
```
>说明：
CMD在Dockerfile中只能出现一次，有多个，只有最后一个会有效。其作用是在启动容器的时候提供一个默认的命令项。如果用户执行docker run的时候提供了命令项，就会覆盖掉这个命令。没提供就会使用构建时的命令。
容器运行时执行的shell命令，CMD指令一般应为Dockerfile文件的最后一行。
例如：`CMD echo $HOME`或者`CMD [ "sh", "-c", "echo $HOME" ]`
举个例子，启动nginx服务不能使用`CMD systemctl start nginx`，而应该使用`CMD ["nginx", "-g", "daemon off;"]`。因为docker会将`CMD systemctl start nginx`命令理解为`CMD ["sh", "-c", "systemctl start nginx"]`，主进程实际上是sh，当systemctl start nginx命令执行完后，sh作为主进程退出了，自然容器也会退出。这就是为什么使用`CMD systemctl start nginx`指令，容器运行不起来的原因。正确的作法是将nginx以前台形式运行，即`CMD ["nginx", "-g", "daemon off;"]`。

### EXPOSE
语法：
```
EXPOSE <port> [<port>...]
```
>说明：告诉Docker服务器容器对外映射的容器端口号，在docker run -p的时候生效。

### ARG
语法：
```
ARG <参数名>   #不设置默认值的话，需要使用--build-arg来设置参数的值
ARG <参数名>[=<默认值>]   #设置参数的默认值
```

>说明：
定义参数，即临时变量，只限于构建镜像时使用，容器运行后是不会存在的。
例如：`ARG APT=apt-get`
注：在构建命令docker build中用--build-arg <参数名>=<值> 可以覆盖此参数的值。
例如：docker build --build-arg APT=yum -t myip:v1 .

### ADD
语法：
```
ADD <源路径>... <目标路径>
ADD ["<源路径1>",... "<目标路径>"]
```

>说明：
复制本机文件或目录或远程文件，添加到指定的容器目录，支持GO的正则模糊匹配。路径是绝对路径，不存在会自动创建。
如果源是一个目录，只会复制目录下的内容，目录本身不会复制。
ADD命令会将复制的压缩文件夹自动解压，这也是与COPY命令最大的不同。
源路径是一个压缩文件的话，将会解压到容器的目标路径下；源路径是一个URL的话，会下载或者解压到容器的目标路径下

### COPY
语法：
```
COPY <源路径>... <目标路径>
COPY ["<源路径1>",... "<目标路径>"]
```
> 说明：
源路径可以是多个，甚至可以使用通配符
COPY除了不能自动解压，也不能复制网络文件。其它功能和ADD相同。

### ENTRYPOINT
语法：
```
ENTRYPOINT <命令>
ENTRYPOINT ["可执行文件", "参数1", "参数2"...]
```

> 说明：
这个命令和CMD命令一样，唯一的区别是不能被docker run命令的执行命令覆盖，如果要覆盖需要带上选项--entrypoint，如果有多个选项，只有最后一个会生效。

### VOLUME
语法：
```
VOLUME <路径>
VOLUME ["<路径1>", "<路径2>"...]
```
>说明：
在主机上创建一个挂载，挂载到容器的指定路径。docker run -v命令也能完成这个操作，而且更强大。这个命令不能指定主机的需要挂载到容器的文件夹路径。但docker run -v可以，而且其还可以挂载数据容器。

### USER
语法：
```
USER daemon
```
> 说明：指定运行容器时的用户名或UID，后续的RUN、CMD、ENTRYPOINT也会使用指定的用户运行命令。

### WORKDIR
语法：
```
WORKDIR path
```
> 说明：为RUN、CMD、ENTRYPOINT指令配置工作目录。可以使用多个WORKDIR指令，后续参数如果是相对路径，则会基于之前的命令指定的路径。如：WORKDIR  /home　　WORKDIR test 。最终的路径就是/home/test。path路径也可以是环境变量，比如有环境变量HOME=/home，WORKDIR $HOME/test也就是/home/test。
### HEALTHCHECK
语法:
```
HEALTHCHECK [选项] CMD <命令>
HEALTHCHECK NONE   #如果基础镜像有健康检查指令，使用这行可以取消健康检查指令
```
>说明：容器健康状态检查指令
例如：`HEALTHCHECK --interval=5s --timeout=3s CMD curl -fs http://localhost/ || exit 1`

HEALTHCHECK的选项有：
```
--interval=<间隔>：两次健康检查的间隔时间，默认为30秒
--timeout=<时长>：每次检查的超时时间，超过这个时间，本次检查就视为失败，默认30秒
--retries=<次数>：指定健康检查的次数，默认3次，如果指定次数都失败后，则容器的健康状态为unhealthy（不健康）
```

### ONBUILD
语法：
```
ONBUILD [INSTRUCTION]
```
>说明：
指定以当前镜像为基础镜像构建的下一级镜像运行的命令
配置当前所创建的镜像作为其它新创建镜像的基础镜像时，所执行的操作指令。
换句话说，就是这个镜像创建后，如果其它镜像以这个镜像为基础，会先执行这个镜像的ONBUILD命令。
