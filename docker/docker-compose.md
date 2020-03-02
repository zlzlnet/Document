# docker-compose

官网：https://docs.docker.com/compose/overview/

## 安装
查找docker-compose版本
https://github.com/docker/compose/releases

获取可执行文件
```
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

修改可执行权限
```
sudo chmod +x /usr/local/bin/docker-compose
```

安装命令联想插件
```
sudo curl -L https://raw.githubusercontent.com/docker/compose/1.23.1/contrib/completion/bash/docker-compose -o /etc/bash_completion.d/docker-compose
```

生效
```
source /etc/bash_completion.d/docker-compose
```

检查
```
[root@localhost ~]# docker-compose version
docker-compose version 1.23.1, build b02f1306
docker-py version: 3.5.0
CPython version: 3.6.7
OpenSSL version: OpenSSL 1.1.0f  25 May 2017
```
