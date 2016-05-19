---
title: kubernetes系列（一）-环境部署
date: 2016-05-19 12:24:05
tag: kubernetes
category : kubernetes
---

kubernetes官方提供了一个[教程](http://kubernetes.io/docs/getting-started-guides/docker-multinode/master/), 可以以docker的形式搭建kubernetes集群，但中间还是会碰到很多坑，相信随着kubernetes的逐渐成熟，这些坑都会被填平

多方摸索之后终于成功搭建了k8s集群，因为要在本地机器部署，这里选择了实用内网＋centos的环境来实现，所有组件均运行在docker环境，k8s官方还提供了多种不同的部署方案，包括：

* [local本机环境](http://kubernetes.io/docs/getting-started-guides/docker/)
* 云环境：
	* [GCE](http://kubernetes.io/docs/getting-started-guides/gce)
	* [AWS](http://kubernetes.io/docs/getting-started-guides/aws)
	* [Azure](http://kubernetes.io/docs/getting-started-guides/coreos/azure/)
	* [CenturyLink Cloud](http://kubernetes.io/docs/getting-started-guides/clc)
* [Docker环境](http://kubernetes.io/docs/getting-started-guides/docker-multinode/master/)

## 各个组件的版本

* centos`7.2` linux `3.10.0`
* docker `1.9.0`
* kubernetes `1.3.0-alpha.3`
* etcd `2.2.1`
* flannel `0.5.5`


## 安装步骤

### 安装docker
这部分比较简单，教程地址：[centos](https://docs.docker.com/engine/installation/linux/centos/)

### 设置环境变量

	export MASTER_IP=<the_master_ip_here>
	export K8S_VERSION=<your_k8s_version (e.g. 1.2.1)>
	export ETCD_VERSION=<your_etcd_version (e.g. 2.2.1)>
	export FLANNEL_VERSION=<your_flannel_version (e.g. 0.5.5)>
	export FLANNEL_IFACE=<flannel_interface (defaults to eth0)>
	export FLANNEL_IPMASQ=<flannel_ipmasq_flag (defaults to true)>

### 启动etcd
kubernetes 使用`flannel`来连接docker daemons之间的网络。Flannel自己也会运行在Docker container中。 为了完成flannel的部署，我们需要自己启动一个docker daemon实例，我们叫它`bootstrap`。我们运行这个docker daemon的时候会加上`--iptables=false` 选项, 所以它只能以`--net=host`的方式启动容器。

执行以下命令来启动bootstrap docker daemon

	sudo sh -c 'docker daemon -H unix:///var/run/docker-bootstrap.sock -p /var/run/docker-bootstrap.pid --iptables=false --ip-masq=false --bridge=none --graph=/var/lib/docker-bootstrap 2> /var/log/docker-bootstrap.log 1> /dev/null &'

启动etcd
	
	sudo docker -H unix:///var/run/docker-bootstrap.sock run -d \
	    --net=host \
	    gcr.io/google_containers/etcd-amd64:${ETCD_VERSION} \
	    /usr/local/bin/etcd \
	        --listen-client-urls=http://127.0.0.1:4001,http://${MASTER_IP}:4001 \
	        --advertise-client-urls=http://${MASTER_IP}:4001 \
	        --data-dir=/var/etcd/data
	       
修改etcd配置文件，设置CIDR范围，注意不要跟已经存在的网络环境冲突
	
	sudo docker -H unix:///var/run/docker-bootstrap.sock run \
	    --net=host \
	    gcr.io/google_containers/etcd-amd64:${ETCD_VERSION} \
	    etcdctl set /coreos.com/network/config '{ "Network": "10.1.0.0/16" }'


### 启动flannel
这时候如果默认的docker已经启动，需要停掉它

	sudo service docker stop
	
然后启动flannel

	sudo docker -H unix:///var/run/docker-bootstrap.sock run -d \
	    --net=host \
	    --privileged \
	    -v /dev/net:/dev/net \
	    quay.io/coreos/flannel:${FLANNEL_VERSION} \
	    /opt/bin/flanneld \
	        --ip-masq=${FLANNEL_IPMASQ} \
	        --iface=${FLANNEL_IFACE}


上面这条命令会输出一长串字符，下面这条命令会用到

	sudo docker -H unix:///var/run/docker-bootstrap.sock exec <really-long-hash-from-above-here> cat /run/flannel/subnet.env
	

### 配置默认docker
这时候我们通过boostrap docker daemon启动了etcd和flannel
可以通过下面这条命令验证

	sudo docker -H unix:///var/run/docker-bootstrap.sock ps
	
	
接下来我们修改docker的启动命令

	sudo vim /lib/systemd/system/docker.service

在`ExecStart`之后增加以下两个参数
	
	--bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU}
	
然后删除docker默认创建的network bridge

	sudo /sbin/ifconfig docker0 down
	sudo brctl delbr docker0

如果brctl命令不存在，就用yum安装一下`bridge-utils`

然后重启docker

	sudo systemctl start docker
	
这时我们`ps aux | grep docker`应该能看到docker daemon启动参数后面多了两个自定义参数

	396 61044 ?       Ssl  May18   8:05 /usr/bin/docker daemon -H fd:// --bip=10.1.1.1/24 --mtu=8972
	

### 启动kubernetes
	sudo docker run \
	    --volume=/:/rootfs:ro \
	    --volume=/sys:/sys:ro \
	    --volume=/var/lib/docker/:/var/lib/docker:rw \
	    --volume=/var/lib/kubelet/:/var/lib/kubelet:rw \
	    --volume=/var/run:/var/run:rw \
	    --net=host \
	    --privileged=true \
	    --pid=host \
	    -d \
	    gcr.io/google_containers/hyperkube-amd64:v${K8S_VERSION} \
	    /hyperkube kubelet \
	        --allow-privileged=true \
	        --api-servers=http://localhost:8080 \
	        --v=2 \
	        --address=0.0.0.0 \
	        --enable-server \
	        --hostname-override=127.0.0.1 \
	        --config=/etc/kubernetes/manifests-multi \
	        --containerized \
	        --cluster-dns=10.0.0.10 \
	        --cluster-domain=cluster.local


`--cluster-dns` 和 `--cluster-domain` 是用来部署dns，如果不需要dns可以去掉

### 验证

下载kubectl工具

	$ wget http://storage.googleapis.com/kubernetes-release/release/v${K8S_VERSION}/bin/linux/amd64/kubectl
	$ chmod 755 kubectl
	$ PATH=$PATH:`pwd`
	
查看所有的nodes

	kubectl get nodes
	
如果一切顺利会得到以下结果

	NAME        LABELS                            STATUS
	127.0.0.1   kubernetes.io/hostname=127.0.0.1   Ready

## 可能会碰到的问题


> 网络无法连接gce的docker repository

这是因为GFW的问题，建议可以自己创建一个私有的docker reposirotry， 找一台有代理的机器下载image下来之后上传到私有repository，只有所有的image都从私有repository pull就行
