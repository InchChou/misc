# docker安装gitlab支持https

首先在云服务器上安装docker，这一步很简单，但是在公司环境下安装可能会有一些坑，这些坑可以见 `docker折腾日志.md`。

## 安装并启用gitlab

官方文档：[GitLab Docker images | GitLab](https://docs.gitlab.com/ee/install/docker.html#gitlab-docker-images)

参考文档：[docker安装GitLab支持http,https_奋斗的小鸟专栏-CSDN博客_docker gitlab https](https://blog.csdn.net/seashouwang/article/details/118994439)

### 拉取gitlab镜像

官网镜像地址：
https://hub.docker.com/r/gitlab/gitlab-ce/tags?page=1&ordering=last_updated

查找GitLab镜像：`docker search gitlab`

拉取gitlab-ce docker最新镜像： `docker pull gitlab/gitlab-ce:latest`

查看本地镜像： `docker images`

### 创建挂载目录

首先在云服务器的家目录中创建一个目录专门来存放跟gitlab相关的目录，主要有`conf`、`data`、`logs`三个目录

```bash
zouyingcheng@localhost:~$ mkdir -p ~/workspace/gitlab/{conf,data,logs}
zouyingcheng@localhost:~$ ls ~/workspace/gitlab/
config  data  logs
```

### 启动容器

```bash
sudo docker run --detach \
  --hostname 10.xxx.xxx.xxx \
  --publish 8443:443 --publish 8080:80 --publish 4222:22 \
  --name gitlab \
  --restart always \
  --volume /home/zouyingcheng/workspace/config:/etc/gitlab:Z \
  --volume /home/zouyingcheng/workspace/logs:/var/log/gitlab:Z \
  --volume /home/zouyingcheng/workspace/data:/var/opt/gitlab:Z \
  --shm-size 256m \
  gitlab/gitlab-ee:latest
```

命令解释：

- `--detach`： 后台运行容器，并返回容器ID
- `--hostname 10.xx.xx.xx`：访问gitlab的主机名或域名
- `--publis 8443:443`：将容器内443端口映射至宿主机8443端口，这是访问gitlab的https端口
- `--restart always`：容器自启动
- `--volume /home/zouyingcheng/workspace/config:/etc/gitlab:Z`：将宿主机上的目录映射到容器内`/etc/gitlab`目录

查看启动日志：`docker logs gitlab`

从日志中可以看到服务启动完成，初始化用户是root，密码在`/etc/gitlab/initial_root_password`文件中。

### 或者使用docker-compose来启动

https://docs.gitlab.com/ee/install/docker.html#install-gitlab-using-docker-compose

## 配置https

### 首先生成证书

```bash
sudo openssl req -new -x509 -days 36500 -nodes -out config/nginx.pem \
         -keyout config/nginx.key -subj "/C=US/CN=gitlab/O=gitlab.com"
```

由于gitlab使用的是crt格式的密钥，所以要将`nginx.pem`改名为`nginx.crt`。然后放入`/home/zouyingcheng/workspace/config/ssl`目录中

```bash
cp config/nginx.crt /home/zouyingcheng/workspace/config/ssl/
cp config/nginx.key /home/zouyingcheng/workspace/config/ssl/
```

### 配置gitlab启用https

这里有两种方式

#### 方式一：修改gitlab.rb文件

https://docs.gitlab.com/omnibus/settings/nginx.html#manually-configuring-https

```bash
vi /etc/gitlab/gitlab.rb

# 下面是文件中需要修改的内容
external_url "https://10.xxx.xxx.xxx:8443"
letsencrypt['enable'] = false
nginx['redirect_http_to_https'] = true
nginx['redirect_http_to_https_port'] = 80
nginx['ssl_certificate'] = "/etc/gitlab/ssl/nginx.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/nginx.key"
nginx['listen_port'] = 80
```

修改完成后，进入容器内部，让配置生效：
`docker exec -it gitlab bash`
在容器内执行:
`gitlab-ctl reconfigure`
`gitlab-ctl restart`

#### 方式二：使用docker-compose设置

创建一个`docker-compose.yml`文件如下：

[正确使用 Docker 搭建 GitLab 只要半分钟 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/49499229)

```yml
version: "3.6"
services:
  gitlab:
    image: gitlab/gitlab-ee:latest
    ports:
      - "4222:22"
      - "8080:80"
      - "8443:443"
    volumes:
      - /home/zouyingcheng/workspace/data:/var/opt/gitlab
      - /home/zouyingcheng/workspace/logs:/var/log/gitlab
      - /home/zouyingcheng/workspace/config:/etc/gitlab
    shm_size: '256m'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
            external_url 'https://10.xx.xx.xx:8443'
            nginx['redirect_http_to_https'] = true
            letsencrypt['enable'] = false
            nginx['ssl_certificate'] = "/etc/gitlab/nginx.crt"
            nginx['ssl_certificate_key'] = "/etc/gitlab/nginx.key"
            # Add any other gitlab.rb configuration here, each on its own line
    secrets:
      - gitlab_root_password
```

最后在该目录下启动 docker：

```bash
sudo docker-compose up --detach --restart always
```

等两分钟，初始化完了，打开浏览器：https://10.xx.xx.xx:8443 即可