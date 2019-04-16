# golang环境配置
goroot
```
/usr/local/go
```

gopath
```
mkdir -p /sure/goproject
cd /sure/goproject
mkdir src pkg bin
```

下载解压
```
wget https://dl.google.com/go/go1.12.3.linux-amd64.tar.gz
tar -C /usr/local -zxvf go1.12.3.linux-amd64.tar.gz
```

编辑环境变量
vim /etc/profile
```
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin
export GOPATH=/sure/goproject
```

## go的环境一个实例
```
go_project     // go_project为GOPATH目录
  -- bin
     -- myApp1  // 编译生成
     -- myApp2  // 编译生成
     -- myApp3  // 编译生成
  -- pkg
  -- src
     -- myApp1     // project1
        -- models
        -- controllers
        -- others
        -- main.go
     -- myApp2     // project2
        -- models
        -- controllers
        -- others
        -- main.go
     -- myApp3     // project3
        -- models
        -- controllers
        -- others
        -- main.go
```
