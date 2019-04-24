
docker logs xxxxxx
```
standard_init_linux.go:207: exec user process caused "exec format error"
```

Q:这个是什么问题?
A:这个很有可能就是image制作的有问题,所以启动的时候起来了又停了,可以看看`docker ps -a`
