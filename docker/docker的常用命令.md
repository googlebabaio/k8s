![](../images/screenshot_1544410864327.png)

示例:
1.启动一个应用，后台运行,命名为`mydevportal`，端口映射为`9527`
```
docker run -d -p 9527:9527 --name mydevportal 192.168.3.27:8888/project/devportal
```

2.批量删除标签为<none>的镜像
```
docker rmi -f `docker images | grep '<none>' | awk '{print $3}'`
```