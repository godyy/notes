为了有效地使用存储驱动程序（storage drivers），重要的是要知道 Docker 如何构建和存储镜像，以及容器如何使用这些镜像。您可以使用这些信息来明智地选择保存应用程序数据的最佳方法，并避免在此过程中出现性能问题。

# 存储驱动程序（storage drivers）对比 Docker Volumes

Docker 使用存储驱动来存储镜像的层（layers），并将数据存储在容器的可写层中。容器的可写层在容器被删除后不会持久化，但适合存储运行时生成的临时数据。存储驱动程序为空间效率进行了优化，但是(取决于对应的驱动)写速度低于本地文件系统性能，特别是对于使用**写时复制**（copy-on-write）文件系统的存储驱动程序。写密集型应用程序(如数据库存储)会受到性能开销的影响，特别是在只读层中存在预先存在的数据时。

Docker 卷用于写密集型数据，必须在容器生命周期之后保存的数据，以及必须在容器之间共享的数据。

|                 | 存储 layer | 空间效率优化 | 写速度低于本地文件系统 | 适合写密集型程序 |
| --------------- | :--------: | :----------: | :--------------------: | :--------------: |
| storage drivers |     是     |      是      |     取决于对应驱动     |        否        |
| docker volumes  |     否     |      否      |           否           |        是        |

# 层（layer） 和 镜像（image） 的关系

Docker image 是由一系列 layer 组成的。每一 layer 代表 image Dockerfile 文件中的一条指令（但不是么条指令都会新建一 layer）。除了最后一 layer 外，每一 layer 都是只读的。

考虑以下 Dockerfile:

```
# syntax=docker/dockerfile:1
FROM ubuntu:18.04
LABEL org.opencontainers.image.authors="org@example.com"
COPY . /app
RUN make /app
RUN rm -r $HOME/.cache
CMD python /app/app.py
```

这个 Dockerfile 包含四个命令。**修改文件系统的命令会创建一个 layer**。`FROM`语句以从`ubuntu:18.04`镜像创建一个 layer 开始。`LABEL`命令只修改图像的元数据，而不生成新 layer。`COPY`命令从 Docker 客户端当前目录添加一些文件。第一个`RUN`命令使用`make`命令构建应用程序，并将结果写入新 layer。第二个`RUN`命令删除一个缓存目录，并将结果写入一个新 layer。最后，`CMD`指令指定在容器中运行什么命令，它只修改镜像的元数据，而不生成新的 layer。

每一 layer 都只是一组与前一 layer 的差异。**注意，添加和删除文件都会产生一个新层**。在上面的例子中，`$HOME/.cache`目录被删除了，但仍可在前一层使用，并加到图像的总大小。

这些 layer 是堆叠在一起的。当您创建一个新的容器时，您将在底层之上添加一个新的可写 layer。这一层通常被称为“容器层”。对正在运行的容器所做的所有更改，如写入新文件、修改现有文件和删除文件，都被写入这个薄可写容器层。下图显示了一个基于`ubuntu:15.04`映像的容器。

![container-layers](../../resource/DevOps/Docker/use-image/container-layers.jpeg)

**存储驱动程序（storage drivers）处理这些层之间交互方式的细节**。不同的存储驱动程序是可用的，它们在不同的情况下有各自的优点和缺点。

# 容器（container）和层（layers）

容器和镜像之间的主要区别在于最上面的可写层。所有向容器中添加新数据或修改现有数据的写操作都存储在这个可写层中。当容器被删除时，可写层也被删除。底层镜像保持不变。

因为每个容器都有自己的可写容器层，所有的更改都存储在这个容器层中，所以多个容器可以共享对相同底层镜像的访问，同时又有自己的数据状态。下图显示了多个容器共享同一个Ubuntu 15.04镜像。

![sharing-layers](../../resource/DevOps/Docker/layer-image-container/sharing-layers.jpeg)

Docker 使用存储驱动程序（storage drivers）来管理镜像的层（layers）和可写容器层的内容。每个存储驱动程序处理实现的方式不同，但所有驱动程序都使用可堆叠的镜像层和写时拷贝(CoW)策略。

# 容器在磁盘上的大小

要查看正在运行的容器的大致大小，可以使用`docker ps -s`命令。两个不同的列与大小有关:

- `size`: 每个容器的可写层使用的(磁盘上的)数据量。
- `virtual size`: 容器使用的只读镜像数据的数据量加上容器的可写层大小。多个容器可以共享部分或全部只读镜像数据。从同一个镜像开始的两个容器共享100%的只读数据，而具有相同镜像的两个容器共享这些公共层。因此，不能只计算虚拟大小的总和。这可能过高估计了总的磁盘使用量。

磁盘上所有正在运行的容器所使用的总磁盘空间是每个容器的`size`和`virtual size`值的某种组合。如果多个容器从相同的镜像开始，那么这些容器在磁盘上的总大小将是 SUM(容器的`size`)加上一个镜像大小(`virtual size`-`size`)。

以上计算并没包括以下容器可占用磁盘空间的其它方式：

- 用于日志驱动程序（logging-driver）存储的日志文件的磁盘空间。如果容器生成大量的日志数据，并且没有配置日志旋转，那么这就不是一件简单的事情。
- 容器使用的数据卷和绑定挂载。
- 用于容器配置文件的磁盘空间，这些配置文件通常很小。
- 写入磁盘的内存(如果启用了交换磁盘)。
- 检查点，如果你正在使用实验性的 检查点/恢复 功能。

# 写时复制（copy-on-write）策略

写时复制是一种共享和复制文件以获得最大效率的策略。如果一个文件或目录存在于镜像的较低层，而另一层(包括可写层)需要对其进行读访问，则它只使用现有的文件。**第一次另一层需要修改文件时(构建镜像或运行容器时)，将文件复制到该层并进行修改**。这将最小化I/O和每个后续层的大小。下面将更深入地解释这些优点。

## 分享推广更小的镜像

当您使用`docker pull`从存储库中拉出一个镜像时，或者当您从本地还不存在的镜像中创建一个容器时，每一层都被分别拉下，并存储在 docker 的本地存储区域中，在 Linux 主机上通常是`/var/lib/docker/`。你可以在这个例子中看到这些层被拉出:

```
$ docker pull ubuntu:18.04
18.04: Pulling from library/ubuntu
f476d66f5408: Pull complete
8882c27f669e: Pull complete
d9af21273955: Pull complete
f5029279ec12: Pull complete
...
```

每一层都存储在 Docker 主机本地存储区域内的自己的目录中。要检查文件系统上的各个层，列出`/var/lib/docker/<storage-driver>`的内容。目录名称与层id不对应。

假设你有两个不同的 dockerfile。你是用第一个去创建名为`acme/my-base-image:1.0`的镜像：

```
# syntax=docker/dockerfile:1
FROM alpine
RUN apk add --no-cache bash
```

第二个基于`acme/my-base-image:1.0`，但有一些附加层：

```
# syntax=docker/dockerfile:1
FROM acme/my-base-image:1.0
COPY . /app
RUN chmod +x /app/hello.sh
CMD /app/hello.sh
```

第二个镜像包含第一个镜像的所有层，加上由`COPY`和`RUN`指令创建的新层，以及一个可读写容器层。Docker 已经有了第一个镜像的所有层，所以它不需要再次拉它们。这两个镜像共享它们所共有的任何层。

如果从两个 dockerfile 构建映像，可以使用`docker image ls`和`docker image history`命令来验证共享层的加密 id 是相同的。

1. 创建一个新的目录`cow-test/`并进入（cd）。

2. 创建一个名为`hello.sh`的新文件，包含以下内容:

   ```
   #!/usr/bin/env bash
   echo "Hello world"
   ```

3. 将上面第一个 Dockerfile 的内容复制到一个名为 Dockerfile.base 的新文件中。

4. 将上面第二个 Dockerfile 的内容复制到一个名为 Dockerfile 的新文件中。

5. 构建第一个镜像。

   ```
   $ docker build -t acme/my-base-image:1.0 -f Dockerfile.base .
   [+] Building 179.2s (10/10) FINISHED
    => [internal] load build definition from Dockerfile.base    						0.1s
    => => transferring dockerfile: 176B                                    0.0s
    => [internal] load .dockerignore                            						0.0s
    => => transferring context: 2B                                      		0.0s
    => resolve image config for docker.io/docker/dockerfile:1       				1.3s
    => docker-image://docker.io/docker/dockerfile:1@sha256:9e2c9ec...     	2.7s
    => => resolve docker.io/docker/dockerfile:1@sha256:9e2c9eca736...     	0.0s
    => => sha256:cb8d6fa06268f199b41ccc0c2718caca915f7... 528B / 528B      0.0s
    => => sha256:b1e2fdbfa8cb359e5e566c70344df9474... 1.21kB / 1.21kB      0.0s
    => => sha256:8d9a8cad598f6c5e97bcb90aab70cd2e8... 9.67MB / 9.67MB      2.1s
    => => sha256:9e2c9eca7367393aecc68795c671f9346... 2.00kB / 2.00kB      0.0s
    => => extracting sha256:8d9a8cad598f6c5e97bcb90aab70cd2e868d3d... 			0.4s
    => [internal] load .dockerignore                                       0.0s
    => [internal] load build definition from Dockerfile.base               0.0s
    => [internal] load metadata for docker.io/library/alpine:latest        0.0s
    => [1/2] FROM docker.io/library/alpine                                 0.0s
    => [2/2] RUN apk add --no-cache bash                                   174.4s
    => exporting to image                                                  0.1s
    => => exporting layers                                                 0.1s
    => => writing image sha256:8925091781b4e0db4978a1f00597501b506... 			0.0s
    => => naming to docker.io/acme/my-base-image:1.0
   ```

6. 构建第二个镜像。

   ```
   [+] Building 1.8s (12/12) FINISHED
    => [internal] load build definition from Dockerfile                    0.0s
    => => transferring dockerfile: 216B                                    0.0s
    => [internal] load .dockerignore                                       0.0s
    => => transferring context: 2B                                         0.0s
    => resolve image config for docker.io/docker/dockerfile:1              0.5s
    => CACHED docker-image://docker.io/docker/dockerfile:1@sha256:...      0.0s
    => [internal] load build definition from Dockerfile                    0.0s
    => [internal] load .dockerignore                                       0.0s
    => [internal] load metadata for docker.io/acme/my-base-image:1.0       0.0s
    => [1/3] FROM docker.io/acme/my-base-image:1.0                         0.0s
    => [internal] load build context                                       0.1s
    => => transferring context: 517B                                       0.0s
    => [2/3] COPY . /app                                                   0.1s
    => [3/3] RUN chmod +x /app/hello.sh                                    0.5s
    => exporting to image                                                  0.1s
    => => exporting layers                                                 0.1s
    => => writing image sha256:198bcd43244b2f7d082929bcdca8e398ac6... 0.0s
    => => naming to docker.io/acme/my-final-image:1.0
   ```

7. 查看每个镜像的大小:

   ```
   $ docker image ls
   
   REPOSITORY						TAG          IMAGE ID       CREATED         SIZE
   acme/my-final-image		1.0          198bcd43244b   3 minutes ago   7.75MB
   acme/my-base-image		1.0          8925091781b4   6 minutes ago   7.75MB
   ```

8. 查看每个镜像的历史:

   ```
   $ docker image history acme/my-base-image:1.0
   
   IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
   8925091781b4   22 minutes ago   RUN /bin/sh -c apk add --no-cache bash # bui…   2.15MB    buildkit.dockerfile.v0
   <missing>      2 months ago     /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
   <missing>      2 months ago     /bin/sh -c #(nop) ADD file:f278386b0cef68136…   5.6MB
   ```

   有些步骤没有大小(0B)，只是元数据更改，不会产生镜像层，也不会占用元数据本身以外的任何大小。上面的输出显示该图像由2个镜像层组成。

   ```
   $ docker image history  acme/my-final-image:1.0
   
   IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
   198bcd43244b   25 minutes ago   CMD ["/bin/sh" "-c" "/app/hello.sh"]            0B        buildkit.dockerfile.v0
   <missing>      25 minutes ago   RUN /bin/sh -c chmod +x /app/hello.sh # buil…   39B       buildkit.dockerfile.v0
   <missing>      25 minutes ago   COPY . /app # buildkit                          222B      buildkit.dockerfile.v0
   <missing>      28 minutes ago   RUN /bin/sh -c apk add --no-cache bash # bui…   2.15MB    buildkit.dockerfile.v0
   <missing>      2 months ago     /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
   <missing>      2 months ago     /bin/sh -c #(nop) ADD file:f278386b0cef68136…   5.6MB
   ```

   注意，第一个镜像的所有步骤也包含在最终的镜像中。最终的镜像包括来自第一个图像的两个镜像层，以及在第二个镜像中添加的两个层。

   > `<missing>`是什么？
   >
   > `docker history`输出中的`<missing>`行表明，这些步骤要么是在另一个系统上构建的，要么是从 Docker Hub 提取的`alpine`映像的一部分，或者是使用 BuildKit 作为构建器构建的。在 BuildKit 之前，“经典”构建器会为每一步生成一个新的“中间”镜像以进行缓存，`IMAGE`列会显示该图像的ID。BuildKit 使用自己的缓存机制，不再需要中间镜像进行缓存。

9. 检查每个镜像的层

   使用`docker image inspect`命令查看每个图像中各层的加密 id:

   ```
   $ docker image inspect --format "{{json .RootFS.Layers}}" acme/my-base-image:1.0
   [
   	"sha256:72e830a4dff5f0d5225cdc0a320e85ab1ce06ea5673acfe8d83a7645cbd0e9cf",
   	"sha256:8001555513b09320e5f352495d7dc00d254e8b8b8ce35c726c748e5bfbf3c047"
   ]
   ```

   ```
   $ docker image inspect --format "{{json .RootFS.Layers}}" acme/my-final-image:1.0
   [
   	"sha256:72e830a4dff5f0d5225cdc0a320e85ab1ce06ea5673acfe8d83a7645cbd0e9cf",
   	"sha256:8001555513b09320e5f352495d7dc00d254e8b8b8ce35c726c748e5bfbf3c047",
   	"sha256:d512a17485dec885763eb16b0ad1ebe45c9fb177dba4ca3796dd1bdb74efe427",
   	"sha256:f631bf376e26a9a4a86fb99bdcb9fa9f993a95a3c777c0471a4e8a75c499d133"
   ]
   ```

   注意，前两个在两个镜像中是相同的。第二个镜像添加了两个额外的层。共享层只在`/var/lib/docker/`中存储一次，并且在将镜像推拉到镜像注册中心时也共享。共享层因此可以减少网络带宽和存储。

## 复制使容器更高效

当您启动一个容器时，将在其他层之上添加一个薄的可写容器层。容器对文件系统所做的任何更改都存储在这里。任何容器没有改变的文件都不会被复制到这个可写层。这意味着可写层越小越好。

当容器中的现有文件被修改时，存储驱动程序执行写时复制操作。涉及的具体步骤取决于特定的存储驱动程序。对于`overlay2`、`overlay`和`aufs`驱动程序，写时拷贝操作遵循以下粗略的顺序:

- 通过镜像层搜搜要更新的文件。这个过程从最新的层开始，然后一层一层地向下工作到基础层。当找到结果时，将它们添加到缓存中以加快未来的操作。
- 对找到的文件的第一个副本执行`copy_up`操作，将文件复制到容器的可写层。
- 任何更改都更新到这个副本中，并且容器不能看到存在于较低层的文件的只读副本。

Btrfs、ZFS 和其他驱动程序以不同的方式处理写时拷贝。

写大量数据的容器比不写大量数据的容器消耗更多的空间。这是因为大多数写操作会在容器的薄可写顶层消耗新的空间。注意，更改文件的元数据，例如，更改文件的权限或所有权，也可能导致`copy_up`操作，因此将文件复制到可写层。

> 提示:对写量大的应用程序使用卷
>
> 对于需要大量写操作的应用程序，不应该将数据存储在容器中。众所周知，应用程序(如写密集型数据库存储)存在问题，特别是在只读层中存在预先存在的数据时。
>
> 相反，使用 Docker 卷，它独立于正在运行的容器，并被设计为对 I/O 高效。此外，卷可以在容器之间共享，并且不会增加容器的可写层的大小。

`copy_up`操作会引起明显的性能开销。这种开销取决于使用的存储驱动程序。大文件、很多层和很深的目录树可以使影响更加明显。每个`copy_up`操作只在第一次修改给定文件时发生，这一事实缓解了这一问题。

为了验证写时复制的工作方式，以下过程基于`acme/my-final-image:1.0`映像自旋了5个容器，并检查它们占用了多少空间。

1. 在 Docker 主机上的终端上运行以下`docker run`命令。末尾的字符串是每个容器的id。

   ```
   $ docker run -dit --name my_container_1 acme/my-final-image:1.0 bash \
     && docker run -dit --name my_container_2 acme/my-final-image:1.0 bash \
     && docker run -dit --name my_container_3 acme/my-final-image:1.0 bash \
     && docker run -dit --name my_container_4 acme/my-final-image:1.0 bash \
     && docker run -dit --name my_container_5 acme/my-final-image:1.0 bash
     
   4f1c74502cf66467533c778a464c8236a794b233cbf2afddf488ecedd3fd47d9
   047ef27d75d1b11cb53a634743092e9a7faf546a290657041e799f4d0e0afba5
   e7f8aebd323fcbcc13743828674a3c277c384a13c6c95dc38fba0abfc1607124
   2fa4cbf1a5c5ee23cd5aeb4e2d0ed22620fd278dff8f96ea801992073b9cd917
   f3a56b7707133a62a2d97a250ddb345b6e31794e5034192c9461914324f65159
   ```

2. 运行带有`--size`选项的`docker ps`命令，以验证5个容器正在运行，并查看每个容器的大小。

   ```
   $ docker ps --size --format "table {{.ID}}\t{{.Image}}\t{{.Names}}\t{{.Size}}"
   
   CONTAINER ID   IMAGE                     NAMES            SIZE
   f3a56b770713   acme/my-final-image:1.0   my_container_5   0B (virtual 7.75MB)
   2fa4cbf1a5c5   acme/my-final-image:1.0   my_container_4   0B (virtual 7.75MB)
   e7f8aebd323f   acme/my-final-image:1.0   my_container_3   0B (virtual 7.75MB)
   047ef27d75d1   acme/my-final-image:1.0   my_container_2   0B (virtual 7.75MB)
   4f1c74502cf6   acme/my-final-image:1.0   my_container_1   0B (virtual 7.75MB)
   ```

   上面的输出显示所有容器共享镜像的只读层(7.75MB)，但是没有向容器的文件系统写入数据，因此没有为容器使用额外的存储空间。

   > 高级:用于容器的元数据和日志存储
   >
   > 注意:这个步骤需要一台 Linux 机器，不能在 Mac 或 Windows 的 Docker Desktop 上工作，因为它需要访问Docker Daemon的文件存储。
   >
   > 虽然`docker ps`的输出提供了容器的可写层所消耗的磁盘空间的信息，但它不包括为每个容器存储的元数据和日志文件的信息。
   >
   > 更多细节可以通过查看 Docker 守护进程的存储位置(默认为`/var/lib/ Docker`)获得。
   >
   > ```
   > $ sudo du -sh /var/lib/docker/containers/*
   > 
   > 36K	/var/lib/docker/containers/047ef27d75d1b11cb53a634743092e9a7faf546a290657041e799f4d0e0afba5
   > 36K	/var/lib/docker/containers/1d7039c1ab2da71947ed23a6146846312554c1cd8db1e6f7aee00aace8e5f012
   > 36K	/var/lib/docker/containers/2fa4cbf1a5c5ee23cd5aeb4e2d0ed22620fd278dff8f96ea801992073b9cd917
   > 36K	/var/lib/docker/containers/32fd34582e253a8c94723bea576a5ed7ec6f44cb15bb4a9a2173bf34434e92c2
   > 36K	/var/lib/docker/containers/4f1c74502cf66467533c778a464c8236a794b233cbf2afddf488ecedd3fd47d9
   > ```

3. 每个容器的存储

   为了演示这一点，运行以下命令将单词' hello '写入容器`my_container_1`, `my_container_2`和`my_container_3`的可写层中的一个文件:

   ```
   $ for i in {1..3}; do docker exec my_container_$i sh -c 'printf hello > /out.txt'; done
   ```

   再次运行`docker ps`命令显示，这些容器现在每个消耗5个字节。该数据对于每个容器都是唯一的，并且不是共享的。容器的只读层不受影响，仍然由所有容器共享。

   ```
   $ docker ps --size --format "table {{.ID}}\t{{.Image}}\t{{.Names}}\t{{.Size}}"
   
   CONTAINER ID   IMAGE                     NAMES            SIZE
   f3a56b770713   acme/my-final-image:1.0   my_container_5   0B (virtual 7.75MB)
   2fa4cbf1a5c5   acme/my-final-image:1.0   my_container_4   0B (virtual 7.75MB)
   e7f8aebd323f   acme/my-final-image:1.0   my_container_3   5B (virtual 7.75MB)
   047ef27d75d1   acme/my-final-image:1.0   my_container_2   5B (virtual 7.75MB)
   4f1c74502cf6   acme/my-final-image:1.0   my_container_1   5B (virtual 7.75MB)
   ```

上面的示例说明了写时复制文件系统如何帮助提高容器的效率。写时复制不仅节省了空间，而且还减少了容器启动时间。当你创建一个容器(或来自同一个映像的多个容器)时，Docker只需要创建薄可写容器层。

如果 Docker 每次创建新容器时都必须复制底层映像堆栈的整个副本，那么创建容器的时间和使用的磁盘空间将显著增加。这类似于虚拟机的工作方式，每个虚拟机有一个或多个虚拟磁盘。vfs存储不提供 CoW 文件系统或其他优化。当使用这个存储驱动程序时，将为每个容器创建映像数据的完整副本。

