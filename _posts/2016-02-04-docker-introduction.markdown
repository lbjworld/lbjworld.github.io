---
layout: post
title: "Docker简介"
categories: system docker
published: true
---

# docker是什么？
docker可谓是近年来虚拟化届的当红明星，各家云平台（AWS，Google Engine等）也都纷纷针对docker虚拟化进行了一定程度的支持，而基于docker的生态圈也是日益丰富：docker-compose，docker-machine，kubernetes，swarm等等，这些工具极大的丰富了docker的使用场景，覆盖了从单机单实例到集群操作的几乎所有的运行场景。那么docker到底是何方神圣呢？我们还是从docker官网的文档引用说起吧：

>Docker allows you to package an application with all of its dependencies into a standardized unit for software development.

也就是说，用户能够方便得将与某一个应用以及相关的依赖打包放到一个标准化单元中，而docker提供的正是生成并在虚拟环境中运行这个标准化单元的工具。


# 与传统虚拟化技术的区别
从技术趋势来说，虚拟化技术大致可以分为三种：完全虚拟化（机器指令级别：VMware、VirtualBox）、准虚拟化（通过修改操作系统底层使其可以运行在容器之上：Xen）、系统虚拟化（由host操作系统本身负责调度管理虚拟化容器，docker正是属于此种类型）。既然同是虚拟化技术，docker与它的老前辈们相比有何优势呢？从下图我们可以直观的看出二者的区别：
![VMs vs. Containers]({{ site.baseurl }}/assets/images/docker-vms-vs-containers.png)

传统的虚拟化技术将VM构建在Hypervisor之上，每个VM几乎又是一套独立的运行体系（包括Guest OS、相关依赖和应用本身），而系统虚拟化如docker，则是以独立的进程空间直接运行在Host OS之上，多个容器共享OS内核资源，同时各自管理自身的应用和相关依赖。笔者认为，相较于其他虚拟化技术，docker的优势在于：

1. docker容器直接运行在Host OS之上，各容器共享系统内核资源，避免了独立虚拟化层的系统开销（如指令集模拟、Guest OS的系统调度等），从而大大提升了系统效率；
2. docker容器以镜像（image）的形式将应用、应用配置和相关依赖进行打包（build），以类似于git管理代码的方式来管理应用运行环境，之后可以进行灵活方便的部署（系统迁移、水平扩展）；
3. docker底层实现直接基于linux内核特性（namespace、cgroup），能够对于容器的运行进行资源限制（CPU核数、内存占用、使用特定用户、配置重启策略等），极大的降低了运维成本；
4. 使用dockerhub对docker镜像进行管理，使得用户能够方便的构建、分享应用，能够以极低的成本快速搭建、测试复杂系统；

任何系统都不是完美的，很多时候系统的长处也许正是其短处所在：

1. 因为直接基于linux内核特性，要想在非linux内核之上运行docker容器，需要额外的虚拟化层（如VirtualBox）运行linux内核（好在大部分的server都使用linux :D）；
2. 以镜像的形式将应用代码、配置和相关依赖全部打包，虽然不必在部署的时候进行各种人工配置，但是打包的镜像往往达到百M（甚至G）级别，而通过dockerhub上传/下载这些镜像则是一个痛苦的过程（尤其是在国内这样特殊的网络环境下:(）；

当然，这些短处在docker社区中也都有其解决之道，笔者将在后续文章中进行详细介绍。


# 技术背景
docker所依赖的技术主要有两类：一类是以内核namespace和cgroup为基础的LXC（LinuXContainer）技术，另一类则是Union File System（例如：aufs、unionfs等）。

## LXC技术

### Linux Namespace

Linux Namespace是Linux提供的一种内核级别环境隔离的方法，提供多种类型的命名空间（类似C++ namespace），使得进程运行时可以在相对隔离的环境中运行，而不受到同时运行的其他进程的干扰。
Linux Namespace 有如下种类，官方文档在[这里](http://lwn.net/Articles/531114/)

| 分类   | 系统调用参数   | 相关内核版本   |
|:--- |:--- |:---:|
| Mount namespaces   | CLONE_NEWNS   | Linux 2.4.19   |
| UTS namespaces   | CLONE_NEWUTS   | Linux 2.6.19   |
| IPC namespaces   | CLONE_NEWIPC   | Linux 2.6.19   |
| PID namespaces   | CLONE_NEWPID   | Linux 2.6.24   |
| Network namespaces   | CLONE_NEWNET   | 始于Linux 2.6.24 完成于 Linux 2.6.29   |
| User namespaces   | CLONE_NEWUSER   | 始于 Linux 2.6.23 完成于 Linux 3.8   |

如果想进一步了解linux namespace在docker中的应用可以参考[这篇文章](http://coolshell.cn/articles/17010.html)

### Linux CGroup

Linux CGroup全称Linux Control Group， 是Linux内核的一个功能，用来限制，控制与分离一个进程组群的资源（如CPU、内存、磁盘输入输出等）。
主要提供了如下功能：

* Resource limitation: 限制资源使用，比如内存使用上限以及文件系统的缓存限制。
* Prioritization: 优先级控制，比如：CPU利用和磁盘IO吞吐。
* Accounting: 一些审计或一些统计，主要目的是为了计费。
* Control: 挂起进程，恢复执行进程。

如果想进一步了解linux cgroup在docker中的应用可以参考[这篇文章](http://coolshell.cn/articles/17049.html)

## Union File System

所谓UnionFS就是把不同物理位置的目录合并mount到同一个目录中。UnionFS的一个最主要的应用是，把一张CD/DVD和一个硬盘目录给联合 mount在一起，然后，你就可以对这个只读的CD/DVD上的文件进行修改（当然，修改的文件存于硬盘上的目录里）。而docker正是基于这个特性实现了其最具特色的镜像（image）管理功能。如下图所示，docker将不同的环境配置封装为不同的blob（在docker内部每个镜像单元叫做blob，这个命名和git中的每个提交单元相同:P），然后在运行时根据这些镜像的层次顺序将他们依次以只读的形式mount到同一个目录中，最后在所有层次之上再mount上一个可读写的目录（即container层），这样在底层运行的进程就可以“看到”一个包含所有环境配置的目录结构，并且可以在运行时对这些目录进行读写操作（这些操作只影响container层，并不会修改只读镜像层）。通过这种巧妙的方式，docker实现了不同镜像之间的复用。

如果想进一步了解AUFS在docker中的应用可以参考[这篇文章](http://coolshell.cn/articles/17061.html)

![Union File System]({{ site.baseurl }}/assets/images/docker-filesystems-multilayer.png)

# Docker系统构成

docker主要由Go语言实现，其系统架构主要分为两个部分：服务端daemon进程和命令式客户端。

*   服务端daemon进程：实现了主要的管理逻辑（镜像管理，运行时container管理，以及内部meta信息的管理操作），一般在每个物理主机上需要运行至少一个docker daemon进程。
*   命令式客户端：即docker命令，做为用户接口，用户使用docker命令对docker环境进行操作。

    例如：`docker ps`可以罗列出当前主机上正在运行的所有container信息，如果要查看一个镜像的详细信息则使用`docker inspect <镜像名称>`。(更多的docker命令请参考[文档](https://docs.docker.com/engine/reference/commandline/cli/))

这里需要特别说明的是docker对于镜像的命名规则：

> \[\<docker registry地址\>/\]\<镜像名称\>\[:\<tag\>\]

* docker registry地址：在docker系统中，我们可以将生成的镜像通过docker push命令推送到指定的镜像管理站点，以方便分享和后续在其他主机上的部署（其他主机可以通过docker pull将镜像下载到本地），docker registry地址即是这个镜像管理站点的地址，docker官方提供的镜像管理地址为[`https://hub.docker.com/`](https://hub.docker.com/)，如果在引用镜像时省略了地址部分，则默认引用官方的docker registry地址，当然docker也支持搭建私有的docker registry（有一个官方的docker镜像提供了这样的环境[registry](https://hub.docker.com/_/registry/)）
* 镜像名称：即是该镜像的名字，很多常用的组件都有其官方的镜像（例如：ubuntu、debian、python、nginx、tomcat、redis、mysql、postgres、elasticsearch等）
* tag：即是该镜像的标记，同一个镜像名下可以有多个tag，而每一个tag分别对应一个镜像。如果省略tag，则默认为`lastest`


# 应用场景

docker适用于几乎所有与环境配置有关的使用场景，下图是一个典型的docker应用场景：

![docker application]({{ site.baseurl }}/assets/images/docker-application-workflow.png)

该图所描述的场景如下：

1. push
    将应用代码从开发环境push到远端中心git
2. pull
    Jenkins系统将代码从中心git pull到build主机
3. unit test
    调用预先准备好的docker单元测试环境镜像运行单元测试，这样能够保证每次运行单元测试的环境的一致性
4. build container
    在通过单元测试之后，由Jenkins自动运行应用container
5. int. test
    对运行中的应用container进行集成测试
6. push webapp
    通过所有测试之后，build应用生产环境镜像并push到私有registry中
7. ssh
    通过ssh登录生产环境
8. pull webapp
    在生产环境通过registry pull需要部署的应用镜像，并进行部署（通过这种方式进行部署，registry中将保存所有版本的生产环境镜像，在紧急情况下，可以很方便的进行回滚操作）

上述整个过程实现了从开发环境、测试环境到生产环境的自动化部署，而在第3个步骤之后，docker的使用成为其自动化过程的重要一环。

# docker-compose

> Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a Compose file to configure your application’s services. Then, using a single command, you create and start all the services from your configuration.

正如官方的介绍所说，docker-compose是一个定义并运行多个docker容器的工具，compose文件以.yml格式（[关于yaml格式](http://www.yaml.org/spec/1.2/spec.html)）定义。

直接上例子吧：

{% highlight YAML %}
rproxy:
    image: "nginx:1.9"
    ports:
     - "5000:80"
    volumes:
     - ./nginx.conf:/etc/nginx/nginx.conf:ro
    links:
     - web:web
web:
    image: "demo_stage3"
    volumes:
     - ./dj_demo/dj_demo/docker_settings.py:/code/dj_demo/settings.py
    links:
     - db:database
    command: ./entry-point.sh
db:
    image: "mysql:5.6"
    environment:
    - MYSQL_ROOT_PASSWORD=password
    - MYSQL_DATABASE=demo
{% endhighlight %}

以上即一个docker-compose.yml文件的全部内容，下面我们来看看这个文件到底都定义了哪些内容：

* 从yaml格式清晰的层次结构中，我们可以看出在整个文件中定义了三个container类型：`rproxy`、`web`、`db`
* 每个container类型下都有若干属性定义，如：image、ports、volumes、links、command、environment等
    * image：指定当前container启动时所使用的镜像名称
    * ports：指定container的网络端口映射（`5000:80`的含义是将container内部的80端口映射到运行主机的5000端口上，这样在主机上访问5000端口即会转发至container内部的80端口）
    * volumes：将主机本地的目录或文件映射到container内部文件系统中（`./nginx.conf:/etc/nginx/nginx.conf:ro`即是将主机当前目录下的nginx.conf配置映射到目标container中的/etc/nginx/nginx.conf，`ro`表示read-only只读映射）
    * links：用于将同一个compose文件中的不同container进行连接（`db:database`表示将以`db`为名称的container连接到当前container中，连接后在当前container中的别名为`database`）
    * command：在container启动时需要执行的命令
    * environment：在container启动后需要附加在container内部的环境变量（一般通过环境变量来对container进行动态配置）

整个docker-compose配置的环境如下图所示：

![docker compose sample]({{ site.baseurl }}/assets/images/docker-compose-sample.png)

之后在命令行执行
{% highlight Bash shell scripts %}
$ docker-compose up
{% endhighlight %}
docker-compose工具将在当前目录下自动找到以`docker-compose.yml`命名的文件并运行。

