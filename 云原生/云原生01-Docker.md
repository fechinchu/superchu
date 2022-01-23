# 云原生01-Docker01

# 1.Docker架构

![image-20211124105436770](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211124105436770.png)

## 1.1.Docker隔离原理

Docker的隔离都是基于Linux系统的

### 1.1.1.资源隔离(namespace)

![image-20211124211003502](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211124211003502.png)

### 1.1.2.资源限制(cgroups)

![image-20211124211054563](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211124211054563.png)

# 2.Docker安装

https://docs.docker.com/engine/install/centos/

该文档有一定问题,在安装指定版本时候:

```shell
# 安装指定版本，用上面的版本号替换<VERSION_STRING>
sudo yum install docker-ce-<VERSION_STRING>.x86_64 docker-ce-cli- <VERSION_STRING>.x86_64 containerd.io
#例如:
#sudo yum install docker-ce-3:20.10.5-3.el7.x86_64 docker-ce-cli-3:20.10.5- 3.el7.x86_64 containerd.io
#注意加上 .x86_64
```

## 2.1.Docker的可视化界面Portainer的安装

```shell
# 服务端部署,访问9000端口即可
docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce

# 代理端部署(用于集群的slave节点)
docker run -d -p 9001:9001 --name portainer_agent --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v /var/lib/docker/volumes:/var/lib/docker/volumes portainer/agent
```



# 3.Docker常见命令

https://docs.docker.com/engine/reference/commandline/docker/

![image-20211125092454567](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211125092454567.png)

 

| 命令      | 作用                                                         |
| --------- | ------------------------------------------------------------ |
| attach    | 绑定到运行中容器的标准输入,输出,以及错误流(这样似乎能够进入容器内容,但是他们操作的就是控制台,控制台的退出命令会让容器退出,比如redis,nginx); |
| build     | 从一个Dockerfile文件构建镜像;                                |
| commit    | 把容器提交创建一个新的镜像;                                  |
| cp        | 容器和本地文件系统之间复制文件/文件夹;                       |
| create    | 创建新容器,但不启动;                                         |
| diff      | 检查容器中文件系统结构的更改;(A:添加文件或目录;D:文件或者目录删除;C:文件或者目录更改;) |
| events    | 获取服务器的实时事件;                                        |
| exec      | 在运行时的容器内运行命令                                     |
| export    | 导出容器的文件系统为一个tar文件.commit是直接提交成镜像,export是导出成文件方便传输; |
| history   | 显示镜像的历史                                               |
| images    | 列出所有镜像                                                 |
| import    | 导入tar的内容创建一个镜像,导入进来的镜像直接启动不了容器;需要使用`docker ps --no-trunc`看下之前的完整命令再用`/docker-entrypoint.sh nginx -g 'daemon off;'`; |
| info      | 显示系统信息                                                 |
| inspect   | 获取docker对象的底层信息                                     |
| kill      | 杀死一个或者多个容器;                                        |
| load      | 从tar文件加载镜像;                                           |
| login     | 登录Docker registry                                          |
| logout    | 退出Docker registry                                          |
| logs      | 获取容器日志;                                                |
| pause     | 暂停一个或者多个容器;                                        |
| port      | 列出容器的端口映射                                           |
| ps        | 列出容器;                                                    |
| pull      | 从registry下载一个image 或者repository;                      |
| push      | 给registry推送一个image或者repository;                       |
| rename    | 重命名一个容器;                                              |
| restart   | 重启一个或者多个容器;                                        |
| rm        | 移除一个或者多个容器;                                        |
| rmi       | 移除一个或者多个镜像                                         |
| run       | 创建并启动容器                                               |
| save      | 把一个或者多个 镜像 保存为tar文件                            |
| search    | 去docker hub寻找镜像                                         |
| start     | 启动一个或者多个容器                                         |
| stats     | 显示容器资源的实时使用状态                                   |
| stop      | 停止一个或者多个容器                                         |
| tag       | 给源镜像创建一个新的标签，变成新的镜像                       |
| top       | 显示正在运行容器的进程                                       |
| unpause   | pause的反操作                                                |
| update    | 更新一个或者多个docker容器配置                               |
| version   | Show the Docker version information                          |
| container | 管理容器                                                     |
| image     | 管理镜像                                                     |
| network   | 管理网络                                                     |
| volume    | 管理卷                                                       |

```shell
# docker的帮助信息,这个很关键;
docker [options] --help

# 删除全部镜像
docker rmi -f $(docker images -aq)
# 移除游离镜像(dangling:游离镜像,没有镜像名字的镜像);
docker image prune

# 重新打标签
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]

# 强制关闭
docker kill [CONTAINER...]
# 允许优雅停机,当前正在运行的程序处理完所有事情后再停止
docker stop [CONTAINER...]

# 特权方式进入容器
docker exec -it --privileged [CONTAINER] /bin/bash

# -------------------commit tag push------------------------------
# 0.提交生成新的镜像
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
# 例如:
docker commit -a zhuguoqing -m 'first commit' mynginx mynginx:v01
# 1.把新的镜像放到远程dockerhub,首先得登录dockerhub
docker login
# 1.1.可以通过`cat ~/.docker/config.json`来获取有没有auth的值来知道有没有登录;
# 2.在dockerhub网页中创建仓库如fechinchu/mynginx
# 3.修改tag使名称与dockerhub的仓库保持一致
docker tag mynginx:v01 fechinchu/mynginx:v01
# 4.推送image
docker push fechinchu/mynginx:v01

# 我们可以使用阿里云的镜像仓库,具体内容参考下图:
# 阿里云链接:https://cr.console.aliyun.com/repository/cn-hangzhou/fechinchu/my_dockerio/details

# -----------------------export import---------------------------
# 导出
docker export mynginx > mynginx.tar
# 导入(会报错,提示'Error response from daemon: No command specified')
docker create --name mynginx -p 80:81 mynginx:v02
# 需要知道之前的启动命令(通过`docker ps --no-trunc`进行查看)
docker create --name mynginx02 -p 80:81 mynginx:v02 /docker-entrypoint.sh nginx -g 'daemon off;'

# ------------------------save load-----------------------------
# save
docker save -o mynginx:v02.tar mynginx:v02
# load
docker load mynginx:v02.tar
```

![image-20211125173040061](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211125173040061.png)

# 4.Docker存储

## 4.1.镜像和容器如何存储

文档:https://docs.docker.com/storage/storagedriver/

### 4.1.1.镜像分层

如下是nginx镜像的分层:

![](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211125193047387.png)

`docker image inspect nginx`获取到镜像的分层:

![image-20211125193224475](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211125193224475.png)

* `LowerDir`:底层目录:diff(只是存储不同),包含小型linux和装好的软件;

  * 小Linux系统+Dockerfile的每一个命令可能都引起了系统的修改,所以和git一样,只记录变化;

  ```shell
  /var/lib/docker/overlay2/8c97cb48d33ff77ca43a0db9d09e807e0dab2cd87ba73bb4310eed0ab9fc161f/diff:用户文件;
  /var/lib/docker/overlay2/e47b262f77df8ec7465c1754166f0e11c4f46c87a25bc9ef4db6b743dac99128/diff:nginx的启动命令;
  /var/lib/docker/overlay2/2a66d356dd46e0ddd801e1cbdd4e52faba3134ac21dae32cd634912269c215a7/diff:
  /var/lib/docker/overlay2/b671d99f6df427bda76026c12c02b5e34297e64d282e0cf39ab3cce349d889e8/diff:nginx的配置文件;
  /var/lib/docker/overlay2/df0f94704f623d47360f3d05826b03267523fda2089e0bda114c6d62dc5c8f3f/diff:小linux系统;
  ```

  * 我们进入到镜像的容器和data目录中查看目录记录inode发现inode都是一直的,可以确定镜像启动的容器,容器的文件系统就是镜像的;

  ![image-20211125195343622](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211125195343622.png)

  ![image-20211125195434234](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211125195434234.png)

  * `docker ps -s`:可以查看容器真正用到的大小

![image-20211125195741731](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211125195741731.png)

* `MergedDir`:合并目录:
* `UpperDir`:上层目录:
* `WorkDir`:工作目录(临时层):

### 4.1.2.容器分层

`docker inspect mynginx`:获取到容器的分层

![image-20211125201227142](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211125201227142.png)

```shell
"LowerDir": "/var/lib/docker/overlay2/5002bb0cf3a07c3cb41a648fbeaecada37d5201584eeda5923d8f8825a3af80f-init/diff:
/var/lib/docker/overlay2/881a11106c95001de16689c5e11128629e22ef076eaa2b7a34e447a049970b1a/diff:
/var/lib/docker/overlay2/8c97cb48d33ff77ca43a0db9d09e807e0dab2cd87ba73bb4310eed0ab9fc161f/diff:
/var/lib/docker/overlay2/e47b262f77df8ec7465c1754166f0e11c4f46c87a25bc9ef4db6b743dac99128/diff:
/var/lib/docker/overlay2/2a66d356dd46e0ddd801e1cbdd4e52faba3134ac21dae32cd634912269c215a7/diff:
/var/lib/docker/overlay2/b671d99f6df427bda76026c12c02b5e34297e64d282e0cf39ab3cce349d889e8/diff:
/var/lib/docker/overlay2/df0f94704f623d47360f3d05826b03267523fda2089e0bda114c6d62dc5c8f3f/diff",
                "MergedDir": "/var/lib/docker/overlay2/5002bb0cf3a07c3cb41a648fbeaecada37d5201584eeda5923d8f8825a3af80f/merged",
                "UpperDir": "/var/lib/docker/overlay2/5002bb0cf3a07c3cb41a648fbeaecada37d5201584eeda5923d8f8825a3af80f/diff",
                "WorkDir": "/var/lib/docker/overlay2/5002bb0cf3a07c3cb41a648fbeaecada37d5201584eeda5923d8f8825a3af80f/work"
```

我们在容器中修改了配置文件,然后在容器UpperDir层目录中找到了该修改的配置文件,有修改的内容,但在镜像的文件中却没有修改的内容;Docker 采用了Copy On Write的机制;

### 4.1.3.Copy On Write

* 写时复制是一种共享和复制文件的策略，可最大程度地提高效率;
* 如果文件或目录位于映像的较低层中，而另一层(包括可写层)需要对其进行读取访问，则它仅使用现有文件;
* 另一层第一次需要修改文件时(在构建映像或运行容器时)，将文件复制到该层并进行修改。 这样可以将I / O和每个后续层的大小最小化;

### 4.1.4.Overlay Driver

Overaly目前是Docker目前默认的驱动;

![image-20211126202211670](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211126202211670.png)

 Merged:合并目录,容器最终的完整工作目录全内容都在合并层;数据卷在容器层产生;所有的增删改都在容器层;

# 5.Docker挂载

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211126202955123.png" alt="image-20211126202955123" style="zoom: 67%;" />

每一个容器支持三种挂载方式

* volumes(卷):存储在主机文件系统的一部分中,该文件系统是由Docker管理(在Linux上是`/var/lib/docker/volumes/`).非Docker进程不应该修改文件系统的这一部分.卷是在Docker中持久存储数据的最佳方法;
* Bind Mount(绑定挂载):可以在任何地方存储在主机系统上,可以随时修改;
* tmpfs mounts(临时挂载):仅存储在主机系统的内存(不建议使用);

## 5.1.volume

```shell
# 匿名卷,docker将创建出匿名卷,并保存容器/etc/nginx下面的内容
docker run -dP -v :/etc/nginx nginx --name mynginx nginx

# 具体卷,docker将创建出名为nginx的卷,并保存容器/etc/nginx下面的内容
docker run -dP -v nginx:/etc/nginx nginx --name mynginx nginx

docker run -dP -v html:/usr/share/nginx/html --name mynginx nginx
#上述操作与下面的操作一样
docker create volume nginxhtml
docker run -dP -v nginhtml:/usr/share/nginx/html --name mynginx nginx
```

通过`docker inspect [containerId]`可以看到挂载目录

![image-20211126233122127](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211126233122127.png)

`/var/lib/docker/volumes`是卷的根目录;

## 5.2.bind mount

```shell
# 所有的宿主机目录是以/开头的都是bind mount
docker run -dP -v /my/nginx:/etc/nginx:ro nginx
```

如果将绑定安装或非空卷安装安装到存在某些文件或目录的容器中的目录,那么这些文件或目录将会被安装遮盖;

# 6.Dockerfile

https://docs.docker.com/engine/reference/builder/

一般而言:Dockerfile可以分为四部分

* 基础镜像信息;
* 维护者信息;
* 镜像操作指令;
* 启动时执行指令;

| 指令       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| FROM       | 指定基础镜像                                                 |
| MAINTAINER | 指定维护者信息,已经过时,可以使用LABEL maintalner = xxx来代替 |
| RUN        | 运行命令v                                                    |
| CMD        | 指定启动容器时默认的命令v                                    |
| ENTRYPOINT | 指定镜像的默认入口,运行命令v                                 |
|            |                                                              |
|            |                                                              |
|            |                                                              |
|            |                                                              |
|            |                                                              |
|            |                                                              |
|            |                                                              |
|            |                                                              |
|            |                                                              |
|            |                                                              |
|            |                                                              |
|            |                                                              |
|            |                                                              |
|            |                                                              |

