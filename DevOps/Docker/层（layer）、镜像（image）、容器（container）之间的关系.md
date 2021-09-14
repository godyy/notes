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

存储驱动程序（storage drivers）处理这些层之间交互方式的细节。

## 关于 Dockerfile

最新的[官方文档](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#minimize-the-number-of-layers)中提到，在老版本的 Docker 中，只有指令`RUN`, `COPY`, `ADD`创建 layer。其他指令创建临时中间 image，并且不增加构建的大小。

# 容器（container）和层（layers）

容器和镜像之间的主要区别在于最上面的可写层。所有向容器中添加新数据或修改现有数据的写操作都存储在这个可写层中。当容器被删除时，可写层也被删除。底层镜像保持不变。

因为每个容器都有自己的可写容器层，所有的更改都存储在这个容器层中，所以多个容器可以共享对相同底层镜像的访问，同时又有自己的数据状态。下图显示了多个容器共享同一个Ubuntu 15.04镜像。

![sharing-layers](../../resource/DevOps/Docker/layer-image-container/sharing-layers.jpeg)

