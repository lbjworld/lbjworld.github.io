---
layout: post
title: "Docker入门"
published: false 
---

在[Docker简介]({% post_url 2016-02-04-docker-introduction %})中，我们对于Docker的技术实现和应用场景有了大体的了解，
但是纸上谈兵终不是我们的目的，关键还是需要将Docker应用到实际的生产项目中。
本文的目标是：

* 在一个干净的系统环境中安装Docker
* 通过几个简单的demo来学习一些基本的Docker应用


# Docker安装
---

Ubuntu 14.04版本系统为例（更多的安装方式请参考[官方文档](https://docs.docker.com/engine/installation/)）进行安装演示。

### 准备工作

> 注意：Docker要求在64位操作系统运行（32位系统请绕行）

Docker的最新版本(1.10.0)要求的内核版本为3.10或以上，内核版本过低将会缺少一些Docker所依赖的特性，或者引发bug导致数据丢失和运行失败。
好吧，让我们首先来确认一下内核版本：

{% highlight Bash shell scripts %}
$ uname -r
3.16.0-51-generic
{% endhighlight %}

输出结果的前面两位即是我们需要确认的版本号，当前版本为3.16，明显高于最低版本要求，如果显示的版本低于3.10，需要先对系统内核进行升级。

> 笔者的Ubuntu系统进行过升级，这里显示的版本略高，对于不同的主机这里显示的版本未必完全一样，只要高于最低版本要求即可。

### 更新APT源

Docker的APT repository包含Docker 1.7.1及以上的版本，我们可以通过以下步骤来设置更新源

> 以下运行要求有root权限

* 更新包信息，确保APT可以使用https方法，并且CA证书已经安装

{% highlight Bash shell scripts %}
$ sudo apt-get update
$ sudo apt-get install apt-transport-https ca-certificates
{% endhighlight %}

* 添加新的GPG key

{% highlight Bash shell scripts %}
$ sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
{% endhighlight %}

* 打开文件`/etc/apt/sources.list.d/docker.list`（如果不存在则新建之），删除之前的所有内容，添加如下内容

{% highlight Bash shell scripts %}
deb https://apt.dockerproject.org/repo ubuntu-trusty main
{% endhighlight %}

* 再次更新源并且移除之前可能安装的老版本

{% highlight Bash shell scripts %}
$ sudo apt-get update
$ sudo apt-get purge lxc-docker
{% endhighlight %}

* 验证设置是否正确

{% highlight Bash shell scripts %}
$ apt-cache policy docker-engine
{% endhighlight %}
输出如下：
{% highlight Bash shell scripts %}
docker-engine:
  Installed: (none)
  Candidate: 1.10.0-0~trusty
  Version table:
     1.10.0-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.9.1-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.9.0-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.8.3-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.8.2-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.8.1-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.8.0-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.7.1-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.7.0-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.6.2-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.6.1-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.6.0-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.5.0-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
{% endhighlight %}

### 安装Docker

* 安装Docker包

{% highlight Bash shell scripts %}
$ sudo apt-get update
$ sudo apt-get install docker-engine
{% endhighlight %}

* 启动Docker服务进程

{% highlight Bash shell scripts %}
$ sudo service docker start
{% endhighlight %}

* 验证Docker是否安装成功

{% highlight Bash shell scripts %}
$ sudo docker run hello-world
{% endhighlight %}

稍加等待之后（Docker将从官方registry下载hello-world:latest镜像并运行）可以获得输出

{% highlight Bash shell scripts %}

Hello from Docker.
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker Hub account:
 https://hub.docker.com

For more examples and ideas, visit:
 https://docs.docker.com/userguide/

{% endhighlight %}

至此Docker安装完成。

### 将用户添加到docker用户组

docker的运行需要root权限，每次都需要在命令前添加`sudo`语句使用起来相当不便，
我们可以将用户添加到docker用户组中以免去输入`sudo`语句。

{% highlight Bash shell scripts %}
$ sudo usermod -aG docker <用户名>
{% endhighlight %}

> 注意：这条语句需要重新登陆终端才能生效

> 注意：docker用户组的权限等同于root用户组，在生产环境中这样做可能会导致安全问题!


# 搭建Web服务
---
下面我们将通过一系列例子来学习Docker的使用，本部分所对应的代码可以在[demo](https://github.com/lbjworld/demo.git)下载。

### Stage 1: 生成Web服务镜像

> 本部分对应docker_demo代码中的[stage 1](https://github.com/lbjworld/demo/tree/master/docker_demo/stage1)

这里我们使用[python](https://www.python.org/) web框架[Django](https://www.djangoproject.com/)来开发web服务，使用的python版本是2.7
（对于Django不熟悉的朋友可以参考[这里](http://docs.30c.org/djangobook2/)）

> 什么？！你对python不熟悉？好吧，参考[这个教程](http://www.pythondoc.com/pythontutorial27/)

好啦，我们来看看项目目录中都有些什么：

* dj_demo：这是我们预先生成好的django项目目录，此目录基本就是传统项目的部分，如果我们愿意，通过修改一些配置完全可以让这个web服务在主机的python环境下运行（此处我们略过不讲）
* Dockerfile：这个部分的重点，docker通过这个Dockerfile文件来生成镜像
* build.sh：为了构建操作方便，写了一个工具脚本
* docker-compose.yml：docker-compose的配置文件，在[Docker简介]({% post_url 2016-02-04-docker-introduction %})中已经略有涉及


#### 构建镜像

首先让我们来看看Dockerfile的内容：

{% highlight Bash shell scripts %}
FROM python:2.7
MAINTAINER "Bijia Lan <lbj.world@gmail.com>"

# setup env
ADD ./dj_demo /code
WORKDIR /code
RUN pip install -r requirements.txt

# run task
CMD ./entry-point.sh
{% endhighlight %}

这是一个很简短清晰的Docker配置文件，文件中每行开头的大写单词都是docker指令，下面我们逐行分析：

* `FROM python:2.7`表示新生成的镜像以`python:2.7`这个镜像为基础，在docker中每个镜像都是基于其他镜像的基础上生成的，
而最底层的系统镜像（比如：`ubuntu:14.04`）则是由官方制作发布的。`FROM`指令就是用于声明当前要构建的镜像是从那个基础镜像继承而来的。
* `MAINTAINER "Bijia Lan <lbj.world@gmail.com>"`用来声明这个镜像的作者信息，`Bijia Lan`就是笔者的大名啦，`<lbj.world@gmail.com>`则是笔者的email（注意要用`<>`括起来）
* `ADD ./dj_demo /code`这个语句是将当前目录（指Dockerfile所在目录）下的dj_demo目录添加到镜像中，同时在镜像中对应的目录为`/code`。
* `WORKDIR /code`该语句将镜像中的`/code`目录设置为工作目录，从该行之后的工作目录切换至`/code`（同时指定了后续运行该镜像时默认的工作目录）。
* `RUN pip install -r requirements.txt`，`RUN`指令用于在构建时执行shell命令，该行语句的作用是通过pip安装我们web应用所需要的python依赖包，这些依赖都罗列在`/code/requirements.txt`中。
* `CMD ./entry-point.sh`，`CMD`指令用于指定镜像启动时所执行的命令行

    > 注意：`CMD`指令所指定的命令行在构建镜像的时候并不会执行，而只在镜像作为容器运行的时候才执行。

到此整个文件解析完毕，我们可以开始构建镜像了。

{% highlight Bash shell scripts %}
$ cd stage1 && ./build.sh
{% endhighlight %}

稍等片刻之后（如果本机没有python:2.7的基础镜像会到官方docker hub下载，可能耗时较长），获得的输出应该是这样的：

{% highlight Bash shell scripts %}
Sending build context to Docker daemon 34.82 kB
Step 1 : FROM python:2.7
 ---> 7a0ad2450c23
Step 2 : MAINTAINER "Bijia Lan <lbj.world@gmail.com>"
 ---> Using cache
 ---> 245daf3a1f76
Step 3 : ADD ./dj_demo /code
 ---> 16d70b7f82dd
Removing intermediate container c2c376ed7129
Step 4 : WORKDIR /code
 ---> Running in 7af83240ff50
 ---> 71be78858c4d
Removing intermediate container 7af83240ff50
Step 5 : RUN pip install -r requirements.txt
 ---> Running in 6b59c9fd9ae7
Collecting django==1.8.3 (from -r requirements.txt (line 1))
  Downloading Django-1.8.3-py2.py3-none-any.whl (6.2MB)
Installing collected packages: django
Successfully installed django-1.8.3
You are using pip version 7.1.2, however version 8.0.2 is available.
You should consider upgrading via the 'pip install --upgrade pip' command.
 ---> 54db7f826884
Removing intermediate container 6b59c9fd9ae7
Step 6 : CMD ./entry-point.sh
 ---> Running in bfa0543787a1
 ---> 27ef334e48dc
Removing intermediate container bfa0543787a1
Successfully built 27ef334e48dc
{% endhighlight %}

可以看到，Dockerfile中的每一行对应一个Step，我们可以通过`docker images`查看刚才构建的镜像

{% highlight Bash shell scripts %}
REPOSITORY                                   TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
demo_stage1                                  latest              27ef334e48dc        3 minutes ago       701.3 MB
{% endhighlight %}

刚才我们只是简单的运行了build.sh脚本，现在让我们来看看整个脚本里做了什么。

{% highlight Bash shell scripts %}
#!/bin/bash

set -ue

sudo docker build -t demo_stage1 .
{% endhighlight %}

除了设置bash脚本运行环境`set -ue`外，这里的有效命令只有一行`sudo docker build -t demo_stage1 .`：

* `docker build`就是用来从Dockerfile构建镜像的命令。
* `-t demo_stage1`指定我们的镜像名称是`demo_stage1`，这里没有指定相应的tag，所以最终构建出的结果中TAG一栏是默认的`lastest`（见之前`docker images`的输出）。
* 注意不要漏掉最后的`.`，这里代表构建过程的工作目录是当前目录，也就是说`docker build`命令会在当前目录下搜索名称为`Dockerfile`的文件并进行构建操作。


#### 启动容器

现在终于到了运行镜像的时候了，不过在运行之前，我们先来看看剩下的那一个文件`docker-compose.yml`

{% highlight YAML %}
# stage 1 docker-compose.yml
web:
    image: "demo_stage1"
    ports:
     - "5000:8000"
    command: ./entry-point.sh 
{% endhighlight %}

这里定义了服务`web`:

* 使用的镜像即是我们刚才生成的`demo_stage1`
* 将内部的8000端口映射到主机的5000端口上
* 运行执行工作目录（还记得Dockerfile中的`/code`吗？）下的`entry-point.sh`脚本

运行如下命令：

{% highlight Bash shell scripts %}
$ docker-compose up
{% endhighlight %}

> 注意：这是一个前台运行命令，可以使用Ctrl+C终止运行

获得的输出大致是这样的：

{% highlight Bash shell scripts %}
Creating stage1_web_1
Attaching to stage1_web_1
web_1 | No changes detected
web_1 | Operations to perform:
web_1 |   Synchronize unmigrated apps: staticfiles, messages
web_1 |   Apply all migrations: admin, contenttypes, auth, sessions
web_1 | Synchronizing apps without migrations:
web_1 |   Creating tables...
web_1 |     Running deferred SQL...
web_1 |   Installing custom SQL...
web_1 | Running migrations:
web_1 |   Rendering model states... DONE
web_1 |   Applying contenttypes.0001_initial... OK
web_1 |   Applying auth.0001_initial... OK
web_1 |   Applying admin.0001_initial... OK
web_1 |   Applying contenttypes.0002_remove_content_type_name... OK
web_1 |   Applying auth.0002_alter_permission_name_max_length... OK
web_1 |   Applying auth.0003_alter_user_email_max_length... OK
web_1 |   Applying auth.0004_alter_user_username_opts... OK
web_1 |   Applying auth.0005_alter_user_last_login_null... OK
web_1 |   Applying auth.0006_require_contenttypes_0002... OK
web_1 |   Applying sessions.0001_initial... OK
{% endhighlight %}

其中以`web_1 | `开头的那些是容器内部的标准输出内容，可以看到进行Django migrate的效果。

打开浏览器输入`http://localhost:5000/hello/`

> It is now 2016-02-05 07:50:32.293994.

Bingo~! web服务构建完成。

> 刷新几次网页，同时观察终端窗口的输出，可以看到从浏览器发送的请求:)


#### 容器运行时

下面我们来看看这个镜像运行时的情况，

> 镜像是磁盘存在形式，运行中的镜像我们称之为容器（Container），这个概念类似于可执行文件和进程的区别。

打开一个新的终端，执行命令：

{% highlight Bash shell scripts %}
$ docker ps
{% endhighlight %}

获得输出：

{% highlight Bash shell scripts %}
CONTAINER ID        IMAGE               COMMAND              CREATED             STATUS              PORTS                    NAMES
3bcc880b800b        demo_stage1         "./entry-point.sh"   29 minutes ago      Up 29 minutes       0.0.0.0:5000->8000/tcp   stage1_web_1
{% endhighlight %}

我们可以看到有一个正在运行中的容器`stage1_web_1`

* `CONTAINER ID`是一个用于唯一标识容器的sha256值的前段截取，通过这个值可以引用相应容器
* `IMAGE`是当前容器运行所基于的镜像名称
* `COMMAND`是容器运行的命令
* `CREATED`是容器创建时间
* `STATUS`是容器状态，这里表示该容器已经运行了29分钟了
* `PORTS`容器的端口映射情况，`0.0.0.0:5000`为主机端口，`8000/tcp`为容器内部端口
* `NAMES`顾名思义就是容器的名字，同样可以通过这个名字来引用相应容器


### Stage 2: 添加数据库支持

> 本部分对应docker_demo代码中的[stage 2](https://github.com/lbjworld/demo/tree/master/docker_demo/stage2)

web服务一般都是以提供数据为目标的，单单运行一个简单的报时web服务是没有太大意义的，本节中我们考虑给之前的web服务添加数据库支持。
数据库是以另一个容器的形式提供的，所以这次我们会同时运行两个容器。

项目目录内容和之前stage1一样

* dj_demo：这是我们预先生成好的django项目目录，这次我们会关注一个Django配置数据库的细节
* Dockerfile：和之前一样，没有变化
* build.sh：和之前一样，没有变化
* docker-compose.yml：主要变化在这里，我们新添加了一个数据库服务

这次我们将主要讨论变化的部分，首先来看看`docker-compose.yml`

{% highlight yaml %}
# stage 2 docker-compose.yml
web:
    image: "demo_stage2"
    volumes:
     - ./dj_demo/dj_demo/docker_settings.py:/code/dj_demo/settings.py
    links:
     - db:db
    ports:
     - "5000:8000"
    command: ./entry-point.sh 

db:
    image: "mysql:5.6"
    environment:
     - MYSQL_ROOT_PASSWORD=password
     - MYSQL_DATABASE=demo
{% endhighlight %}

对比之前的文件，我们添加了一个叫做`db`的服务：

* 这个服务基于`mysql:5.6`（该镜像详细的配置参考[这里](https://hub.docker.com/_/mysql/)）镜像构建，这里我们直接使用官方构建好的镜像了
* `environment`设置了一些容器运行时的环境变量，一会在查看Django数据库配置的时候会用到

而对于原有的`web`服务，我们也做了相应的改变：

* 为了避免重复，这里将镜像重新构建并命名为`demo_stage2`
* `volumes`将主机目录下的Django配置文件映射到容器内部的对应位置，方便动态修改配置
* `links`将`db`服务容器链接到当前容器中，并以`db`作为其别名

下面我们来看看Django中对于数据库部分的相应配置，Django的配置文件位于`dj_demo/dj_demo/docker_settings.py`，这里我们只列出和数据库相关的部分：

{% highlight python %}
# Database
# https://docs.djangoproject.com/en/1.8/ref/settings/#databases

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': os.getenv('DB_ENV_MYSQL_DATABASE'),
        'USER': 'root',
        'PASSWORD': os.getenv('DB_ENV_MYSQL_ROOT_PASSWORD'),
        'HOST': 'db', # DB hostname
        'PORT': '3306',
    }
}
{% endhighlight %}

我们看到Django在运行时从环境变量中获取数据库的名称和数据库的密码，这两个环境变量是从哪里来的呢？
我想大家应该已经可以猜到了，这两个环境变量是从链接到`web`容器的`db`容器提供的（我们在docker-compose.yml文件中可是给`db`容器指定了两个环境变量哦）。
不过这里有一点问题，我们先前指定的环境变量是`MYSQL_ROOT_PASSWORD`和`MYSQL_DATABASE`，为什么在配置中引用的是`DB_ENV_MYSQL_ROOT_PASSWORD`和`DB_ENV_MYSQL_DATABASE`呢？
原来这里是有一个映射规则的，当一个容器被链接到另一个容器中后，其原有的环境变量就会被添加上一个链接别名的前缀和一个固定的ENV前缀：`<链接别名>_ENV_<原有环境变量名称>`
此例中链接数据库的别名是`db`，所以我们得到以`DB_ENV_`开头的两个环境变量。

这里还有一个细节需要注意，即我们将数据库的host设置为`db`，这个也是和链接别名有关系，docker会自动将如下这样一个条目添加到容器内的hosts文件中：

{% highlight Bash shell scripts %}
10.1.36.3   db
{% endhighlight %}

这样我们就可以通过别名来直接引用数据库容器了，前面的那个IP是docker内部分配给数据库容器的IP（不同的机器或者每次运行的时候都可能不同）。
 

### Stage 3: 添加反向代理

> 本部分对应docker_demo代码中的[stage 3](https://github.com/lbjworld/demo/tree/master/docker_demo/stage3)

# 关于Docker的一些运维工具
---


