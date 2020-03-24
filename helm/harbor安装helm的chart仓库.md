1.安装 harbor chart的仓库

```
docker-compose stop
./install.sh --with-chartmuseum
```

2.安装helm chart的插件
```
helm plugin install https://github.com/chartmuseum/helm-push
```

如果网络不好，可以先去下载再解压
```
wget https://github.com/chartmuseum/helm-push/releases/download/v0.8.1/helm-push_0.8.1_linux_amd64.tar.gz


tar zxf helm-push_0.8.1_linux_amd64.tar.gz
mkdir -p /root/.local/share/helm/plugins/helm-push
chmod +x bin/*
mv bin plugin.yaml /root/.local/share/helm/plugins/helm-push
```


安装完成后，可以看到：
```
# helm plugin list
NAME    VERSION DESCRIPTION
push    0.8.1   Push chart package to ChartMuseum
```


3.在harbor上创建chart的仓库
随便建立一个项目，比如叫做 `jidanjuan`

4.在helm上将harbor的chart仓库添加
```
helm repo add chartmuseum http://192.168.3.6:8888/chartrepo/jidanjuan --username=admin --password=Harbor12345
```

查看新添加的chart仓库
```
# helm repo list
NAME    URL
aliyun  https://apphub.aliyuncs.com/
myrepo  http://192.168.3.6:8888/chartrepo/jidanjuan
```

5.将chart上传
```
helm push mysql-6.8.0.tgz --username=admin --password=Harbor12345 http://192.168.3.6:8888/chartrepo/jidanjuan
```

输出：
```
Pushing mysql-6.8.0.tgz to http://edgehub.acedge.cn:8888/chartrepo/jidanjuan...
Error: 401: Unauthorized
Error: plugin "push" exited with error
```

待解决。。
