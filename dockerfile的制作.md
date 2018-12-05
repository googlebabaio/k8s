## build命令
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


> 注：在构建镜像时，Docker daemon会首先将Dockerfile所在的目录构建成一个context（上下文），然后通过Dockerfile里的COPY或者ADD语句将context里的文件复制到指定的镜像目录里。所以需要复制到镜像里的文件，无论是脚本、安装包还是配置文件，都需要跟Dockerfile文件放在同一个目录下。
当执行docker build命令后，返回的第一条信息便是在构建上下文。

## Dockerfile的基本指令
基本指令有十多个，分别是：
* FROM
* MAINTAINER
* RUN
* CMD
* EXPOSE
* ENV
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
用法：`FROM <image>`
说明：第一个指令必须是FROM了，其指定一个构建镜像的基础源镜像，如果本地没有就会从公共库中拉取，没有指定镜像的标签会使用默认的latest标签，可以出现多次，如果需要在一个Dockerfile中构建多个镜像。

### MAINTAINER
用法：`MAINTAINER <name> <email>`
说明：描述镜像的创建者，名称和邮箱

### RUN
用法：
```
RUN <command>
RUN ["executable", "param1", "param2" ...]
```

说明：RUN命令是一个常用的命令，执行完成之后会成为一个新的镜像，这里也是指镜像的分层构建。一句RUN就是一层，也相当于一个版本。这就是之前说的缓存的原理。我们知道docker是镜像层是只读的，所以如果第一句安装了软件，用完在后面一句删除是不可能的。所以这种情况要在一句RUN命令中完成，可以通过&符号连接多个RUN语句。RUN后面的必须是双引号不能是单引号（没引号貌似也不要紧），command是不会调用shell的，所以也不会继承相应变量，**要查看输入RUN "sh" "-c" "echo" "$HOME"，而不是RUN "echo" "$HOME"。**

### CMD
用法：
```
CMD <命令>
CMD ["可执行文件", "参数1", "参数2"...]
```
说明：CMD在Dockerfile中只能出现一次，有多个，只有最后一个会有效。其作用是在启动容器的时候提供一个默认的命令项。如果用户执行docker run的时候提供了命令项，就会覆盖掉这个命令。没提供就会使用构建时的命令。
容器运行时执行的shell命令，CMD指令一般应为Dockerfile文件的最后一行。
例如：CMD echo $HOME或者CMD [ "sh", "-c", "echo $HOME" ]
举个例子，启动nginx服务不能使用CMD systemctl start nginx，而应该使用CMD ["nginx", "-g", "daemon off;"]。因为docker会将CMD systemctl start nginx命令理解为CMD ["sh", "-c", "systemctl start nginx"]，主进程实际上是sh，当systemctl start nginx命令执行完后，sh作为主进程退出了，自然容器也会退出。这就是为什么使用CMD systemctl start nginx指令，容器运行不起来的原因。正确的作法是将nginx以前台形式运行，即CMD ["nginx", "-g", "daemon off;"]。

### EXPOSE
用法：`EXPOSE <port> [<port>...]`
说明：告诉Docker服务器容器对外映射的容器端口号，在docker run -p的时候生效。

### ENV
用法：
```
ENV <key> <value>   #设置一个环境变量
ENV <key1>=<value1> <key2>=<value2>...   #设置多个环境变量
```

> 说明：设置容器的环境变量，可以让其后面的RUN命令使用，容器运行的时候这个变量也会保留。

### ARG
```
ARG <参数名>   #不设置默认值的话，需要使用--build-arg来设置参数的值
ARG <参数名>[=<默认值>]   #设置参数的默认值
```

说明：定义参数，即临时变量，只限于构建镜像时使用，容器运行后是不会存在的。
例如：`ARG APT=apt-get`
注：在构建命令docker build中用--build-arg <参数名>=<值> 可以覆盖此参数的值。
例如：docker build --build-arg APT=yum -t myip:v1 .

### ADD
用法：`ADD <src>   <dest>`
>说明：复制本机文件或目录或远程文件，添加到指定的容器目录，支持GO的正则模糊匹配。路径是绝对路径，不存在会自动创建。如果源是一个目录，只会复制目录下的内容，目录本身不会复制。ADD命令会将复制的压缩文件夹自动解压，这也是与COPY命令最大的不同。

### COPY
用法：`COPY <src> <dest>`
> 说明：COPY除了不能自动解压，也不能复制网络文件。其它功能和ADD相同。

### ENTRYPOINT
用法：
```
ENTRYPOINT <命令>
ENTRYPOINT ["可执行文件", "参数1", "参数2"...]
```

> 说明：这个命令和CMD命令一样，唯一的区别是不能被docker run命令的执行命令覆盖，如果要覆盖需要带上选项--entrypoint，如果有多个选项，只有最后一个会生效。

### VOLUME
用法：`VOLUME ["path"]`
> 说明：在主机上创建一个挂载，挂载到容器的指定路径。docker run -v命令也能完成这个操作，而且更强大。这个命令不能指定主机的需要挂载到容器的文件夹路径。但docker run -v可以，而且其还可以挂载数据容器。

### USER
用法：`USER daemon`
> 说明：指定运行容器时的用户名或UID，后续的RUN、CMD、ENTRYPOINT也会使用指定的用户运行命令。

### WORKDIR
用法：`WORKDIR path`
> 说明：为RUN、CMD、ENTRYPOINT指令配置工作目录。可以使用多个WORKDIR指令，后续参数如果是相对路径，则会基于之前的命令指定的路径。如：WORKDIR  /home　　WORKDIR test 。最终的路径就是/home/test。path路径也可以是环境变量，比如有环境变量HOME=/home，WORKDIR $HOME/test也就是/home/test。

### ONBUILD
用法：`ONBUILD [INSTRUCTION]`
> 说明：配置当前所创建的镜像作为其它新创建镜像的基础镜像时，所执行的操作指令。意思就是，这个镜像创建后，如果其它镜像以这个镜像为基础，会先执行这个镜像的ONBUILD命令。