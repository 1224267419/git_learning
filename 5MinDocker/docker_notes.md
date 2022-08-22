# 容器架构

容器由两部分组成：程序与依赖。

容器与VM都是封装，隔离。区别在于：

<img src="docker_notes.assets\1659102918291.png" alt="1659102918291" style="zoom: 50%;" />

带来什么好处：

- 轻量，启动速度快
- 各种服务、程序能在不同的环境中顺利部署且运行

## 架构详解

![1659103320349](docker_notes.assets\1659103320349.png)

核心组件：

- 客户端 Client
  - docker命令 或 REST API
- 服务器 Docker daemon
  - 运行于host的后台，负责创建、运行、监控容器，构建、存储镜像
- 镜像 Images
  - 能够通过镜像创建容器
- 容器 Container
  - 镜像运行的实例
- Registry
  - 镜像仓库

## 安装、运行试试

主机上安装docker host的步骤有：

- 安装HTTPS CA证书

  ```bash
  $ sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
  ```

- 添加GPG key

  ```bash
  $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
  ```

- 添加Docker的apt源

  ```bash
  $ sudo add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) \
    stable"
  ```

- 安装docker

  ```bash
  $ sudo apt-get update
  $ sudo apt-get install docker-ce
  ```

运行试试：

```
docker run -d -p 80:80 httpd	# 部署httpd容器（若没有则自动从DockerHub下载镜像）
								# -d daemon后台启动, -p 将容器80端口映射到Host的80端口
docker images	# 查看已经下载的镜像
docker ps		# 显示正在运行的容器
```

# 镜像

## 镜像分层结构

### 镜像层

我们可以在Host上构建不同的base镜像 (base image)，即其他镜像可以以之为基础进行扩展的镜像（一般是各种Linux发行版）。而且由于各个Linux发行版之间主要的区别是在**rootfs**上，kernel并无大差别，因此都一致直接使用Host的kernel。（若对kernel有要求，不建议使用容器，可能VM更适合）

然后在base之上我们可以构建更多的镜像，每安装一个软件，就在镜像上再增加一层。这样带来的一个好处就是**共享资源**。

![1659384064749](docker_notes.assets\1659384064749.png)

### 容器层

在容器启动时，一个可写容器层（writable container）会被加载到镜像顶部。他的特别之处是：只有容器层可写，其以下的镜像层都**只读**。通过容器层，我们可以实现Copy-on-Write：

- Docker分层的特点：当存在相同路径的文件时，会把下层的复制到容器层，再将容器层的文件**覆盖下层的文件**。
- 那么通过容器层，我们可以实现对镜像文件的增加/修改/删除，但是他们其实**只发生在容器层**，并不影响到镜像。

### 缓存

在构建镜像层（`docker build`）或下载镜像（`docker pull`）的时候，如果**包括该层以前的内容已经执行过**（镜像层已存在），则会直接使用缓存中的镜像层来构建。如果不想使用缓存，请在指令后方加入 `--no-cache`。

但由于上层是依赖于下层的，如果中间有丝毫变化，都会使上面所有层的缓存失效。

原镜像：`FROM a -> RUN b`

可以使用缓存b：`FROM a -> RUN b -> RUN c` 

不可以使用缓存b：`FROM a -> RUN c -> RUN b `

## 构建镜像

原始的方法是通过 `docker commit`

1. 交互模式 `-it` 运行容器
2. 修改容器（比如安装软件）
3. `docker ps`：找到默认的新容器名
4. `docker commit default-name new-image-name`

但这种方法过于原始，且效率低，容易出错，并不推荐使用，因此**Dockerfile**成为了潮流。

### Dockerfile

利用这一行语句，可通过在当前目录中寻找对应**Dockerfile**来部署镜像：

```bash
docker build -t new-image-name .
```

此语句会引导docker将build context（即当前目录）的文件发送给docker daemon以便后续`ADD`或`COPY`等操作。因此不要把多余的文件放在context里减缓构建速度。

此后可以通过`docker history image-name`来查询具体的构建/分层过程。

### 镜像命名

命名时实际上由两部分组成：`[repository]:[tag]`，（更准确的说，完整格式是`[registry-host]:[port]/[username]/[repository]:[tag]`，但Docker Hub上的镜像可以省略主机和端口号）若没有标明[tag]则一律使用默认值`latest`。

而一个镜像可以使用`docker tag`设置多个tag，那么便方便于版本迭代：

> 比如 myimage 目前为 v1.9.1，我们可以给他打上 latest, 1, 1.9, 1.9.1 四个tag。等更新到 v1.9.2了，给他打上 latest, 1, 1.9, 1.9.2，前面三个tag是从原来1.9.1的移过来的。等更新 v2.0.0了，给他打上了 latest, 2, 2.0, 2.0.0。那么保证了 myimage:1 始终指向 1.x.x最新镜像，myimage: 1.9指向1.9.x最新镜像。

当我们想删除镜像的时候，如果一个镜像不止一个tag，那么当执行`docker rmi [repository]:[tag]` 的时候，只是单纯untag了这一个tag，只有最后一个tag被删除，镜像才会完整被删除。

### Dockerfile 常用指令

| 指令          | 作用                                                         |
| ------------- | ------------------------------------------------------------ |
| FROM          | 指定 base 镜像                                               |
| MAINTAINER    | 指定镜像作者                                                 |
| COPY src dest | 从 build context 复制到镜像                                  |
| ADD src dest  | 同上，但若src是压缩文件，会自动解压到dest                    |
| ENV           | 设置环境变量                                                 |
| EXPOSE        | 暴露某个端口                                                 |
| VOLUMN        | 详情看*存储章节*                                             |
| WORKDIR       | 为此后的RUN, CMD, ADD, COPY, ENTRYPOINT设置工作目录          |
| RUN           | 在容器中运行指定指令                                         |
| CMD           | 容器启动时默认运行指定的命令（可有多个，但只有最后一个有效） |
| ENTRYPOINT    | 容器启动时默认运行指定的命令（可有多个，但只有最后一个有效） |

**RUN, CMD, ENTRYPOINT 都有两种格式**

- shell：RUN \<command\>
  - 在这种情况下，如果先前有定义变量，command里$变量会被代替
- exec：RUN ["executable", "param1", ...]
  - 不会解析变量

**CMD, ENTRYPOINT 的异同**

- CMD：在docker run没有参数的时候才会执行
- ENTRYPOINT：无论如何都会执行，而且CMD可以提供额外参数

如：

```dockerfile
ENTRYPOINT ["/bin/echo", "Hello"]
CMD ["World"]
```

如果 `docker run -it [image]`则输出 Hello world（world为默认补充的）

如果 `docker run -it [image] User`则输出 Hello user（cmd被忽略）

**最佳实践**

- RUN 用于安装应用
- 若要运行程序/服务，则使用ENTRYPOINT Exec，CMD可以作为默认参数
- 默认启动的命令就用CMD

## Registry

### Docker Hub

> Docker Hub 地址：https://hub.docker.com/

1. `docker login -u username`
2. `docker push new-image:[tag]`

（当要区分不同用户的同名镜像时，可以用`docker tag`在镜像名字前面加上用户名：`[username]/[repository]:[tag]`）

（如果要上传同一镜像的全部版本的话，忽略不写[tag]）

### 本地 Registry

> 官方文档：https://docs.docker.com/registry/configuration

简而言之就是需要启动一个registry容器，将容器目录映射到Host的指定目录来存放镜像数据，参照上方链接。

# 容器

> 如何运行容器在前面已经有提及过了。可以用`docker run image-name`运行，也可以在后面增加命令来指定运行容器时的命令。

- 加上`-d`便可以让其后台运行。
- 加上`-it`便可以在容器启动后直接进入。
- 加上 `--name` 可以指定容器名称，也可以之后用`docker rename`重命名。

## 进入容器

1. `docker attach 长ID`
2. `docker exec -it <container> bash` 或 `sh`

他们之间的区别：

- `attach`直接进入容器，且使用长ID
- `exec`是在容器中打开新终端，并且可以启动新进程，且可以使用短ID（即长ID的前十二位）
- 如果只是想看容器的输出的话，用`docker attach`或者`docker logs`都可以

## 容器状态

![1659707568965](\docker_notes.assets\1659707568965.png)

### 重启/停止容器

- `docker stop` 向容器进程发送`SIGTERM`
- `docker kill` 向容器进程发送`SIGKILL`
- `docker start` 启动被停止的docker
- `docker restart` 就是 stop + start

对于服务类的容器，启动时就可以设置`--restart=always`来让他无论何种原因退出都重启服务。

### 暂停容器

- `docker pause`
- `docker unpause`

### 删除容器

- `docker rm`

## 容器限制

### 内存

1. `-m` 或 `--memory` 设置内存使用限额
2. ``--memory-swap` 设置 内存+ swap 使用限额

### CPU

1. `-c` 或 `--cpu-shares`，默认为1024（单位是相对权重值）
2. `--cpu` 定义CPU的数量

### Block IO

1. `--device-read-bps`、`--device-write-bps` 限制每秒读写数据量
2. `--device-read-iops`、`--device-write-iops `限制每秒读写iops
3. `--blkio-weight ` 设置读写带宽权重

## 容器底层

### Cgroup

上述对容器的限制，其实都是linux通过**cgroup**进行的。即`/sys/fs/cgroup`

CPU: `/cpu/docker/`

内存: `/memory/docker/`

BlockIO: `/blkio/docker`

### namespace

> 知乎上有一个很好的回答：https://www.zhihu.com/question/28300645/answer/2488146755

让容器看起来拥有了整个文件系统，实际上是**通过namespace实现了资源的隔离**。

- mount namespace：目录隔离
- UTS namespace：hostname隔离
- IPC namespace：信号量隔离
- PID namespace：PID隔离
- Network namespace：网卡，IP，路由隔离
- User namespace：用户隔离

# 网络

## 原生网络

Docker安装时自动创建的三个网络：none, host, bridge（可通过`docker network ls`）查询，在运行容器的时候可以通过 `--network=网络名` 指定。

### none 网络

挂在**none网络**底下的容器，只有本地回环地址。适用于一些安全性要求高、不需要联网的应用。

### host 网络

挂在**host网络**下的容器可以使用host的所有网卡，即直接让容器配置host网络。

### bridge 网络

**bridge网络**是使用最广泛，且默认使用的网络。

在Docker安装时会创建一个名为 `docker0` 的 linux bridge，在容器创建时，docker自动从172.17.0.0/16中分配一个IP给容器使用。



在操作后可通过 `brctl show` 查看 host 的网络结构。也可以通过 `docker network inspect [net_name]`  来查看对应docker配置信息（当然也包括网络）

##  user-defined 网络

可以通过已有的网络驱动(`bridge`,`overlay`,`macvlan`)创建自定义网络。

`docker network create [net_name]`

- `--driver`: 指定驱动
- `--subnet`: 指定子网
- `--gateway`: 指定网关

（在指定了subnet后，才可以在运行容器的时候用 `--ip` 指定容器的静态IP）

## 容器连通性

### 容器间通信

同一个网络内的容器可以相互通信，但不同的网络之间存在着 **DOCKER-ISOLATION** 规范，即不同的网络之间会被隔离，无法通信。此时可以通过给容器添加网卡来进行让该容器能访问另一个网络进行通信。

`docker network connect [net-name] [container-name]`、

### 与外部通信

容器是默认可以**访问外部**的，在访问的时候，会交由给host利用NAT替换其source IP至host的真正拥有的IP地址。

外部网络访问容器，则是通过端口映射执行，比如执行`docker run -d -p 80 httpd`，可通过`docker ps`或`docker port [container]`来查看端口映射状况。

在运行的时候也可以通过指定其映射的host端口，比如：`docker run -d -p 23333:80 httpd`

且对应每一个映射的端口，host都会有一个`docker-proxy`来处理访问容器的流量。

这图就能很好的反映与外部网络通信的情况：

<img src="D:\Programming\Books\docker_k8s\docker_notes.assets\1660512326450.png" alt="1660512326450" style="zoom: 33%;" />

# 存储

## storage driver

回忆起镜像分层结构最上层有一层可写的容器层，他使得镜像和容器的创建、共享、分发变得高效，其实都得益于storage driver——它实现了多层数据堆叠且提供单一的合并后的视图。

storage driver有多种。默认使用的是Linux发行版默认的storage driver。使用`docker info`可查看对应driver。

对于一次性、无状态的数据，使用storage driver维护存储是很方便的选择。但对于需要持久化的数据需求，那就需要其他的机制了。

## Data volume

data volume 实际上是 docker host 文件系统中的目录，只不过被 mount 到容器的文件系统中。

那我们怎么在 storage 和 volume 之间做选择呢？很简单，无状态的软件在容器层，而持久化数据放在volume里。比如：

- 数据库软件 vs 数据
- Web应用 vs 日志
- Apache Server vs HTML文件

有两种方法做volume：

1. **bind mount**

在运行docker容器的时候，加上`-v`指定mount的地址以及权限。

例子：`docker run -d -v <host path>:<container path>:[permission] <container name>`

2. **docker managed volume**

跟 **bind mount** 很相似。只是这次不指定`<host path>`，直接指定`<container path>`。

这是可以通过`docker inspect`来查看他给我们定义的mount source在哪里：

每当申请mount volume的时候，docker自动在/var/lib/docker/volumes生成一个目录作为mount源，且这个mount源会与`<container path>`的内容完全一致（如果`<container path>`存在的话）

## 数据共享

### 容器与host共享

除了volume的方式以外，可以通过`docker cp`的方式将host的文件拷贝到volume里。

### 容器之间共享

方法有三种：

1. 将要共享的数据一同放进对应容器们的bind mount中。
2. **volume container**

首先创建volume container`docker create <vc name> -v <path1> -v <path2> ... `

然后之后的容器运行时全部通过`--volumes-from <vc name>`来直接共享volume container的mount。

3. **data-packed volume container**

该方法将数据完全打包到 volume container 的镜像中，然后通过docker managed volume共享。

Dockerfile中，加上以下两距

```dockerfile
ADD <host source path> <container dest path>
VOLUME <container dest path>
```

然后构建新镜像，以新镜像来运行volume container。此后再让别的容器`--volumes-from`它即可。

## Volume的生命周期

> 关于备份、恢复、迁移、销毁。

### 备份与恢复

**备份**：定期将本地镜像存在host的本地registry中

**恢复**：取出来，拷贝进去即可

### 迁移

如果我们要使用更新的Registry，却不改变volume的情况，则需

- `docker stop`停止使用当前Registry容器
- 启动新版本容器并让他mount到原有的volume上

### 删除

在删除容器时加上`-v`即可。但如果没有加就会产生孤儿 volume，可以通过`docker volume ls`来查看以及`docker volume rm`来删除

# 多主机管理

## Docker Machine 安装与创建

- 

