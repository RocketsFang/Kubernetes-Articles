最近我看到了一条来自让人敬畏的Amycodes关于Kubernetes Pods的tweet信息：
你知道么为什么一个Pod中的容器总是被一起调度？ 因为他们是嵌套的容器
虽然它不是100%准确（多个容器就不是一件事情，我们会进一步了解）但他还是指出了Pods是一件令人咂舌的事情。  总之值得一看究竟Pods和容器是个啥并且学习一下他们
事实上是个啥。

Kubernetes上关于Pods的文档提供了最好的也是最完整的对Pods的解释但是他们写的太常见并且用了很多的行话。 我也是推荐大家去通读它毕竟和我写的相比他是更好更
准且的解释。我这篇文章希望是更接地气的补充

## 容器就不是容器
很多的读者已经知道这个但是Linxu的容器并不是实际存在的。 在Linux世界里面就没有个实体叫做‘容器’。 正如大家知道的容器就是一个利用了Linux两个功能命名空间
和cgrou的一个普通的进程。 命名空间可以让你提供一个进程的视图，这个视图隐藏了进程之外的所有事情， 因此为进程提供了一个他自己的运行环境。 这就可以使这个
特定的进程看不到也不能被其他进程所干扰。

命名空间包括：
* Hostname
* Process IDs
* File System
* Network interfaces
* Inter-Process Communication (IPC)

我说的在命名空间内运行的进程不会被其他的进程干扰这就话并不十分准确。 进程可以利用在所在的物理机器的所有资源， 因此它可以饿死其他争取相同资源的进程。 
Linux上有个特性叫做cgroups。 进程可以在一个cgroup中运行就好像命名空间一样，但是cgroup是用来限制这个进程能够使用的资源的。这些资源包括CPU,RAM，block 
I/O等等。 CPU可以用milli-cores来限制， RAM可以用字节来限制。 进程本身是正常运行的但是他只能使用cgroup限制的CPU以及如果他超过了cgroup设置的内存限制
就会报内存耗尽的错误。

这儿还有很多的学习资源而不仅仅是cgroup和命名空间，google他们我强烈要求你们阅读他们。

这里我要指出的是cgrou和每一个命名空间类型是截然不同的特性。 一些上面列出的命名空间可能会被用到也可能使用不到。 你可能只用了cgroup或者其他两个的组合。
命名空间和cgroup也能用于一组的进程。 在单个的命名空间中你可以运行多个进程。 那样的话他们就能彼此可见就可以之间进行通信。或者你可以在单独的cgroup中运行这组进程，那样的话他们就一起受限于特定数量的CPU和内存。

## 组合中的组合
当你用Docker运行一个普通的容器， Docker会为每一个容器创建命名空间和cgroup，这些设置就一个一个被映射。 这就是开发者思考容器的方式。
![开发者眼中的容器](https://github.com/RocketsFang/Kubernetes-Articles/blob/master/images/containers.png)

容器可以说是个重要的独立的导弹发射仓，但也有些例外就是他也需要数据卷和端口映射到物理主机以使它能够和外界交流。

然而，使用一些额外的命令行参数，你就能够将多个容器放入到一个命名空间当中去。我们下创建一个单独的nginx容器
```
cat <<EOF >> nginx.conf
error_log stderr;
events { worker_connections 1024;}
http{
  access_log /dev/stdout combined
  server{
  listen 80 default_server;
  server_name example.com ww.example.com
  location / {
  proxy_pass http://127.0.0.1:2368;
  }
}
EOF
docker run -d --name nginx -v `pwd`/nginx.conf:/etc/nginx/nginx.conf -p 8080:80 nginx
```
接下来我们在这个容器里面运行一个ghost的容器。 这次我们需要一些额外的参数来是他加入到nginx容器命名空间当中去。
```
docker run -d --name ghost --net=container:nginx --ipc=container:nginx --pid=container:nginx ghost
```
这样我们的nginx容器就能在localhost上代理请求到我们的ghost容器。 如果你访问 http://localhost:8080/ 你应当能够通过nginx代理看到ghost。 这些命令在一套命名空间中创建了一系列的容器。 这些命名空间可以让Docker容器彼此之间相互发现和通信。
![组合中的组合](https://github.com/RocketsFang/Kubernetes-Articles/blob/master/images/ghost_.png)

## Pods就是多个容器（可以这么说）
现在我们看到了我们可以用多个进程来组合命名空间和cgroups， 我们能够清楚的知道Kubernetes Pods到底是个啥了。 Pods允许你指定你想要运行的容器，而Kubernetes帮我们用正确的方式自动设置好了命名空间和cgroups。 实际的实现会比我们的这个例子更加的复杂，比如Kubernetes没有使用Docker的网络而是使用了[CNI](https://github.com/containernetworking/cni)，但这并不妨碍我们理解概念。

当我们以这种方式创建了我们的容器组，每一个进程就感觉好像他们是运行在同一个机器当中一样。 他们可以彼此以localhost的方式通信， 他们可以使用同一个数据卷。他们甚至可以使用IPC或者彼此之间发送诸如HUP或者TERM信号。

我们在设想一下假设我们要运行nginx和confg，并且要利用confg去更新nginx的配置，然后不管什么时候当我添加或者删除应用程序服务器是都要重启nginx。 可以在假设我们还有一个保存了我们后端服务器IP地址的etcd服务器。一旦那个IP地址列表发生了改变confg就会收到一个提醒，confg就写一个新的nginx配置文件并发送一个HUP信号给nginx这样nginx就会去重新加载这个新的配置文件。

![干点儿实际的事儿](https://github.com/RocketsFang/Kubernetes-Articles/blob/master/images/nginx.png)

我们需要这样去做用Docker方式的话，我们需要把nginx和confg应用程序放到一个单独的容器中。 由于Docker只有一个entrypoint，而我们需要运行一个监控进程来管理这两个进程。这样去做似乎不是太理想因为我们的需要给每一个要运行的nginx额外再运行一个监控程序。 更重要的是由于这个监控进程是entrypointDocker只知道这个进程的存在，并不能够感知其他进程的存在就是说你的应用程序或者其他的工具不可能通过Docker API操作这些进程。 Nginx服务器可能都已经崩溃了，可是Docker根本对此一无知晓。

![糟糕的设计](https://github.com/RocketsFang/Kubernetes-Articles/blob/master/images/supervisord.png)

使用Kubernetes的Pods来管理每一个进程进而识别他的状态。 这样一来就可以通过API来得到进程的状态信息并且还能提供诸如当他崩溃后重新启动自动化的日志服务。

![喜欢这个设计](https://github.com/RocketsFang/Kubernetes-Articles/blob/master/images/kubernetes.png)

## API形式的容器Pods化
将多个container组合到Pod里面的可能使得我们可以基本上创建其他Pods可以以API方式消费的容器到一个Pod中。这里的API并不能理解为普通的Web API，更是一个抽象可以被其他Pod使用的抽象。

比如，在我们的nginx+confd例子中confg根本不知道nginx进程的任何情况。 他所知道的就是他的监空在etcd上面的value然后发送HUP信号量给nginx进程或者运行一个命令。 具体的应用程序不一定必须是nginx，它可以是任何的应用程序。 这样的话，你就可以用confg的镜像然后配置到各种pod中去。 Pod就会去在摩托车上调用我们称之为边车。

你也可以思考你可能会用到的其他的抽象。 像istio那样的服务网格就可以当作边车装置来扣到我们的Pod上来为我们提供服务了路由，遥测和策略加强而不需要去修改我们的主程序。

你也可以去使用多个’边车‘。 比如一起同时使用confg边车和istio边车。 应用程序就可以这样去组合来创造出更加复杂稳定的系统，但是还会保持各自程序相对的简单。

希望这篇文章可以给你一些启示，对你将来为什么用Pod以及怎么用Pod去部署你的容器有帮助。


































































## reference
https://www.ianlewis.org/en/what-are-kubernetes-pods-anyway
