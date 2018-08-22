# 万能的Pause容器

当我们查看我们的Kubernetes 集群的节点时， 在节点上运行```docker ps```命令时我们会发现有很多正在运行的Pause容器。
```
$ docker ps
CONTAINER ID        IMAGE                           COMMAND ...
...
3b45e983c859        gcr.io/google_containers/pause-amd64:3.0    "/pause" ...
...
dbfc35b00062        gcr.io/google_containers/pause-amd64:3.0    "/pause" ...
...
c4e998ec4d5d        gcr.io/google_containers/pause-amd64:3.0    "/pause" ...
...
508102acf1e7        gcr.io/google_containers/pause-amd64:3.0    "/pause" ...
```
这些Pause容器是什么？为什么会有这么多？究竟发生了什么？
为了回答这些问题， 我们得往前看看，明白一下在Kubernetes的世界里Pod是怎么被实现的，特别是和DOcker容器运行时。 如果你还是不怎么了解，你可以看看我的之前
关于Kubernetes pods的文章

Docker支持容器，这对于部署单个单元的应用程序十分有价值。 然而但你要一起运行多个这样的单元时这个模型就变的笨重不易执行了。我们会经常看到这样的场景，当
开发人员开发Docker镜像时都会用管理进程作为入口点来启动和管理多个进程。在生产环境中，又会发现去以一组部分隔离环境部分共享环境的容器去部署这些应用会更加
有用。

对于这样的用例Kubernetes提供了一个称为Pod的抽象，它隐藏了设置Docker标志值的复杂性和对容器监控的需求和数据卷的分享等等。他同时也隐藏了容器运行时环境的
差异。比如对rkt的原生支持所以kubernetes只要做很少部分的事情但是作为kubernetes的最终用户你不需要为此感到担忧。

原则上来说，任何人都能够去配置Docker去完成在容器组之间的共享级别-你只需要创建一个父容器，这个如容器知道如何正确的设置标志值并别用这些标志值去创建共享
相同的环境， 然后管理这些组中的容器生命周期。 管理这些容器组的生命周期会变得很难。

在Kubernetes世界中，在你的pod中pause容器就是这个父容器的角色。它有两个核心的责任，第一在Pod中他是共享Linux命名空间的基础。第二，随着进程命名空间的共
享它将是所有进程的父进程(PID 1)进而回收产生的僵尸进程。

## 共享命名空间
在Linux世界中，当你创建了一个进程，它就会从他的父进程中继承命名空间。如果用unsharing的方式创建一个不继承父命名空间的新的进程在新的命名空间这样一来这个
进程就是在一个新的命名空间之中。这有一个使用unshare工具在一个新的命名空间运行一个shell
```
sudo unshare --pid --uts --ipc --mount -f chroot rootfs /bin/sh
```
一但这个process运行起来，你就可以加入其他的process到这个新进程的命名空间来组成一个Pod。新的进程是可以通过setns系统调用加入到一个已经存在的命名空间的

在Pod中的所有容器是共享命名空间的。 Docker提供了一些进程自动化配置的工作下面我们从头使用Pause容器和共享命名空间的例子来创建一个Pod。 首先我们需要启动一
个Puase容器然后我们再加入我们其他的容器到Pod中。
```
docker run -d --name pause -p 8080:80 gcr.oi/goole_container/pause-amd64:3.0
```
然后我们启动其他的Pod容器，首先我们启动一个ngix 容器。 这个容器会创建一个对localhost 2368端口的nginx请求代理。
**注意 我们也同时把Pause容器的8080端口映射到80端口而不是nginx容器，因为Pause容器创建了一个初始网络命名空间ngnix可以加到这个命名空间中去**
```
$cat <<EOF >>nginx.conf
error_log stderr;
events { worker_connections 1024; }
hhtp{
  access_log /dev/stdout combined;
  server{
    listen 80 default_server;
    server_name example.com www.example.com;
    location / {
      proxy_pass http://127.0.0.1:2368;
    }
  }
} 
EOF

docker run -d --name nginx -v `pwd`/nginx.conf:/etc/nginx/nginx.conf --net=container:pause --ipc=container:pause --pid=container:pause 
nginx
```
然后我们将创建另一个容器ghost blog应用程序
```
docker run -d --name ghost --net=container:pause --ipc=container:pause  --pid=container:pause ghost
```
在这两个例子中我们都指定了Pause容器作为我们想要加入的目标命名空间容器。 这样我们就能很快创建一个Pod。 如果你访问http://localhost:8080你就应该能看到
nginx作为代理的ghost博客页面，因为网络命名空间是在Pause，nginx，ghost容器之间共享的。












## Reference
reference: https://www.ianlewis.org/en/almighty-pause-container



































