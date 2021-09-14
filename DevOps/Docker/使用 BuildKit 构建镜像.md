针对 18.09 版本的 Docker Build 增强引入了构建体系结构急需的彻底改革。通过集成 BuildKit，用户可以看到性能、存储管理、特性功能和安全性方面的改进。

- 使用 BuildKit 创建的 Docker 镜像可以推送到 Docker Hub，就像使用`docker build`创建的 Docker 镜像一样
- Dockerfile 格式在`docker build`中也适用于 BuildKit
- 新的`--secret`命令行选项允许用户传递秘密信息，以使用指定的 Dockerfile 构建新镜像

# 限制

- 尽支持构建 linux 容器

# 启用 BuildKit

最简单的方法是，在使用`docker build`命令之前，设置环境变量`DOCKER_BUILDKIT=1`：

```
$ DOCKER_BUILDKIT=1 docker build .
```

或者，设置默认开启，在 Docker 守护进程配置`/etc/docker/daemon.json`中加入如下配置，并重启守护进程：

```
{ "features": { "buildkit": true } }
```

# 新的命令行输出

默认：

```
$ DOCKER_BUILDKIT=1 docker build -t acme/my-base-image:1.0 -f Dockerfile.base .

[+] Building 1.0s (10/10) FINISHED
 => [internal] load build definition from Dockerfile.base   										0.0s
 => => transferring dockerfile: 101B                                            0.0s
 => [internal] load .dockerignore                                               0.0s
 => => transferring context: 2B                                                 0.0s
 => resolve image config for docker.io/docker/dockerfile:1                      0.5s
 => CACHED docker-image://docker.io/docker/dockerfile:1@sha256:9e2c9...         0.0s
 => [internal] load build definition from Dockerfile.base                       0.0s
 => [internal] load .dockerignore                                               0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                0.0s
 => [1/2] FROM docker.io/library/alpine                                         0.0s
 => CACHED [2/2] RUN apk add --no-cache bash                                    0.0s
 => exporting to image                                                          0.0s
 => => exporting layers                                                         0.0s
 => => writing image sha256:8925091781b4e0db4978a1f00597501b5064b07da3d8f51eeff01c004bb91285         0.0s
 => => naming to docker.io/acme/my-base-image:1.0                               0.0s
```



# 覆盖默认的前端

新的`syntax` Dockerfile 特性可用于覆盖默认的前端（Dockerfile 版本）。

```
# syntax=<frontend image>, e.g. # syntax=docker/dockerfile:1.2
```

上面的例子，使用`docker/dockerfile`的版本为 1.2.0 或更高的前端特性。

推荐使用`docker/dockerfile:1`，总是指向`version 1`的最新释出版。BuildKit 在构建之前会自动检查 syntax 更新，确保你使用的是最新版。

# 新的 secret

docker build 的新`--secret`标志允许用户向 Dockerfile 传递秘密信息，以安全的方式构建 docker 镜像，这些信息不回存储在最终镜像中。

`id`是要传递到`docker build --secret`中的标识符。这个标识符与要在 Dockerfile 中使用的`RUN --mount`标识符相关联。Docker 不会在 Dockerfile 之外使用 secret 文件的文件名，因为这可能是敏感信息。

`dst`将 secret 文件重命名为 Dockerfile `RUN`命令中要使用的特定文件。

例如，存储在文本文件中的一段秘密信息：

```
$ echo 'WARMACHINEROX' > mysecret.txt
```

以及使用指定使用 BuildKit 前端`docker/ Dockerfile:1.2`的 Dockerfile，可以在执行`RUN`时访问这个秘密信息:

```
# syntax=docker/dockerfile:1.2

FROM alpine

# shows secret from default secret location:
RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret

# shows secret from custom secret location:
RUN --mount=type=secret,id=mysecret,dst/foobar cat /foobar
```

需要使用`--secret`标志将 secret 传递给 build。这个 Dockerfile 只是为了证明可以访问这个秘密。正如您可以看到的，在构建输出中打印了 secret。最终构建的镜像将没有秘密文件:

```
$ docker build --no-cache --progress=plain --secret id=mysecret,src=mysecret.txt .
...
#7 [2/3] RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret
#7 sha256:75601a522ebe80ada66dedd9dd86772ca932d30d7e1b11bba94c04aa55c237de
#7 0.455 WARMACHINEROX
#7 DONE 0.5s

#8 [3/3] RUN --mount=type=secret,id=mysecret,dst=/foobar cat /foobar
#8 sha256:a1db940558822fcffbe7da0dc8b9f590a2870c01ea3a701051b7ce68412dc694
#8 0.494 WARMACHINEROX
#8 DONE 0.5s
...
```

# 使用 SSH 在构建中访问私有数据

`docker build`有一个`--ssh`选项，允许 docker 引擎转发 SSH 代理连接。有关SSH代理的详细信息，请参阅 [OpenSSH 手册页](https://man.openbsd.org/ssh-agent)。

只有 Dockerfile 中通过定义`type=ssh` mount 显式请求 SSH 访问的命令才能访问 SSH 代理连接。其他命令不知道有任何可用的 SSH 代理。

为了请求 Dockerfile 中的`RUN`命令的 SSH 访问，定义一个类型为 `ssh` 的 mount。这将设置`SSH_AUTH_SOCK`环境变量，使依赖 SSH 的程序自动使用该套接字。

下面是一个在容器里使用 SSH 的 Dockerfile 例子:

```
# syntax=docker/dockerfile:1
FROM alpine

# Install ssh client and git
RUN apk add --no-cache openssh-client git

# Download public key for github.com
RUN mkdir -p -m 0600 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts

# Clone private repository
RUN --mount=type=ssh git clone git@github.com:myorg/myproject.git myproject
```

创建 Dockerfile 之后，使用`--ssh`选项连接 SSH 代理。

```
 $ docker build --ssh default .
```

您可能需要首先运行`ssh-add`来将私钥身份添加到身份验证代理，以使其工作。