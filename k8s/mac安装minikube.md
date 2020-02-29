# mac 安装 minikube

time: 20200229  
minikube version: 1.7.3
kubectl version: 1.17.3

### 更新软件  
```
brew update
```

### 安装虚拟化软件
```
brew install docker-machine-driver-hyperkit

sudo chown root:wheel /usr/local/opt/docker-machine-driver-hyperkit/bin/docker-machine-driver-hyperkit

sudo chmod u+s /usr/local/opt/docker-machine-driver-hyperkit/bin/docker-machine-driver-hyperkit
```

### 下载minikube
```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64 

sudo install minikube-darwin-amd64 /usr/local/bin/minikube

minikube version
```

### 下载kubectl
```
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl"

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl

kubectl version --client
```

### 启动minikube
```
minikube start --registry-mirror=https://registry.docker-cn.com --registry-mirror=https://z4h6ikoc.mirror.aliyuncs.com --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers --iso-url=https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/iso/minikube-v1.7.3.iso --vm-driver="hyperkit"  --cpus=2 --memory=2048

# registry-mirror 镜像加速地址 （必须否则会很慢）
# https://registry.docker-cn.com
# https://z4h6ikoc.mirror.aliyuncs.com 我的阿里云镜像加速器
# image-repository（必须带,重点,重点）
# registry.cn-hangzhou.aliyuncs.com/google_containers
# vm-driver  hyperkit 类型
# iso-url minikube iso的地址
# https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/iso/minikube-v1.7.3.iso
```

如果安装失败
```
minikube delete

cd ~/.minikube
rm -rf *
```

常用命令
```
minikube status

minikube dashboard # 打开控制台
```


## 参考文档
https://minikube.sigs.k8s.io/docs/start/macos/
https://yq.aliyun.com/articles/221687