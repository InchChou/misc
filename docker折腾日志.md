**docker代理相关：**

docker设置http/https代理：

https://docs.docker.com/config/daemon/systemd/#httphttps-proxy

docker容器设置http/https代理：

https://docs.docker.com/network/proxy/#set-the-environment-variables-manually

 

在设置代理的时候，https代理服务器应该以http开头，不然会有TLS handshake timeout等问题，如 https://stackoverflow.com/questions/51571686/ubuntu-18-04-error-response-from-daemon-get-https-registry-1-docker-io-v2 所述

 

**docker影响公司代理：**

在我使用过程中，安装docker后，计算云无法访问公司内网，并且ping代理服务器的时候提示Destination Host Unreachable

https://blog.csdn.net/MC48431354/article/details/101605468

原因：

当 Docker server 启动时，会在主机上创建一个名为 docker0 的虚拟网桥，由此实现容器和外部的网络通信，docker0 默认的容器ip为 172.17.0.1/16，而公司代理的地址也恰好在 172.17.*.* 这个网段，而被误认为是容器内部ip,从而通过docker0尝试访问容器，自然是访问不到的。

 

在使用镜像仓库之前，首先需要docker login，这样才能正常search和pull镜像

 

**docker以非root用户启用：**

https://docs.docker.com/engine/install/linux-postinstall/