<!-- toc -->
## 修改docker默认的数据存放目录
```
systemctl stop docker
cd /var/lib
cp -rf docker docker.bak
cp -rf docker /xxx/
rm -rf docker
ln -s /xxx/docker docker
systemctl start docker
docker info
```
