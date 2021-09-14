`Docker Compose` 是 Docker 官方编排（Orchestration）项目之一，负责快速的部署分布式应用。

# 1. 简介

`Compose` 项目是 Docker 官方的开源项目，负责实现对 Docker 容器集群的快速编排。

`Compose` 定位是 「定义和运行多个 Docker 容器的应用（Defining and running multi-container Docker applications）」，其前身是开源项目 Fig。

我们知道使用一个 `Dockerfile` 模板文件，可以很方便的定义一个单独的应用容器（生成镜像，通过镜像创建容器）。然而，在日常工作中，经常会碰到需要多个容器相互配合来完成某项任务的情况。例如要实现一个 Web 项目，除了 Web 服务容器本身，往往还需要再加上后端的数据库服务容器，甚至还包括负载均衡容器等。

`Compose` 恰好满足了这样的需求。它允许用户通过一个单独的 `docker-compose.yml` 模板文件（YAML 格式）来定义一组相关联的应用容器为一个项目（project）。

## 1.1 概念

`Compose` 中有两个重要的概念：

- 服务（`service`）：一个应用的容器，实际上可以包含若干运行相同镜像的容器实例。
- 项目（`project`）：由一组关联的应用容器组成一个完整的业务单元，在`docker-compose.yml`文件中定义。

`Compose` 的默认管理对象是项目，通过子命令对项目中的一组容器进行便捷地生命周期管理。

`Compose` 项目由 Python 编写，实现上调用了 Docker 服务提供的 API 来对容器进行管理。因此，只要所操作的平台支持 Docker API，就可以在其上利用 `Compose` 来进行编排管理。

## 1.2 特色

- Multiple isolated environments on a single host，单主机多隔离环境

  Compose 使用`project`名称将环境彼此隔离。默认的项目名称是项目所在目录，默认项目目录即 Compose 文件所在目录。可通过`--project-directory`指定项目目录。

  项目名称可通过`-p`标志或`COMPOSE_PROJECT_NAME`环境变量指定。

- Preserve volume data when containers are created，创建容器时保留卷数据

  Compose 保存服务用到的所有卷。当`docker-compose up`运行时，如果它找到了任何之前运行过的容器，它会拷贝旧容器的卷到新容器中。这样保证了你在卷中创建的任何数据都不会丢失。

- Only recreate containers that have changed，仅重新创建已变更的容器

  Compose 缓存创建容器所用配置。当你重启未变更的服务时，Compose 会重用已存在的容器，这样意味着你可以快速变更应用环境。

- Variables and moving a composition between environments，变量和在环境之间移动 composition（一个 Compose 项目）

  Compose 文件支持 variables。可以使用变量来根据不同的环境自定义 composition。

  可以使用`extends`字段或创建多个 Compose 文件来扩展 Compose 文件。

## 1.3 通用场景

- 开发环境。

  开发软件时，在一个隔离环境中运行应用且与之交互的能力至关重要。

  Compose 文件提供了一种方式来记录和配置应用程序的所有服务依赖项（数据库，队列，缓存，web 服务 APIs 等）。使用 Compose 命令行工具可以用一个简单的命令（`docker-compose up`来为每一个依赖项创建和启动一个或多个容器。

- 自动化测试环境

  任何 CD 或 CI 过程的一个重要部分是自动化测试套件。端到端的自动化测试需要环境来运行测试。

  Compose 提供了一种方便的方法来为你的测试套件创建和销毁隔离的测试环境。通过在 Compose 文件里定义完整的环境，只需要几个命令就可以创建和销毁这些环境。

  ```
  docker-compose up -d # 启动
  ./run-tests # 运行测试套件
  docker-compose down # 销毁
  ```

- 单机部署

  Compose 在传统上专注于“开发”和“测试流程“，但是随着版本的释出，也在更多产品向的特性上取得了进展。

# 2. 安装与卸载

参照官方文档安装与卸载：https://docs.docker.com/compose/install

安装命令行补全：https://docs.docker.com/compose/completion/

# 3. 使用

使用 Compose 基本上有3个步骤：

1. 用`Dockerfile`定义应用程序环境，这样就可以在任何地方重现。
2. 在`docker-compose.yml`文件中定义组成应用的服务，这样它们就可以一起运行于一个隔离的环境中。
3. 执行`docker-compose up`命令来启动和运行整个应用。

下面已一个 web 网站应用项目来举例。

## 基于`python`实现的一个能够记录访问次数的 web 网站

### 1. 设置项目结构

1. 为项目创建目录。

   ```
   mkdir composetest
   cd composetest
   ```

2. 在项目目录中创建`app.py`文件并复制下面代码：

   ```
   import time
   
   import redis
   from flask import Flask
   
   app = Flask(__name__)
   cache = redis.Redis(host='redis', port=6379)
   
   def get_hit_count():
       retries = 5
       while True:
           try:
               return cache.incr('hits')
           except redis.exceptions.ConnectionError as exc:
               if retries == 0:
                   raise exc
               retries -= 1
               time.sleep(0.5)
   
   @app.route('/')
   def hello():
       count = get_hit_count()
       return 'Hello World! 该页面已被访问 {} 次.\n'.format(count)
   ```

   在代码中，`redis`是在应用所在隔离环境网络中的 redis 容器的主机名。我们使用默认的 redis 端口，6379。

3. 创建`requirements.txt`文件并复制：

   ```
   flask
   redis
   ```

### 2. 设置镜像环境

为构建镜像编写 Dockerfile。镜像中包含 python 应用程序需要的所有依赖项，包括 python 自己。

在项目目录中创建`Dockerfile`并复制如下：

```
# syntax=docker/dockerfile:1
FROM python:3.7-alpine
WORKDIR /code
ENV FLASK_APP=app.py
ENV FLASK_RUN_HOST=0.0.0.0
COPY . .
RUN apk add --no-cache gcc musl-dev linux-headers \
	&& pip install -r requirements.txt
EXPOSE 5000
CMD ["flask", "run"]
```

### 3. 创建 Compose 文件，定义 services

创建`docker-compose.yml`文件，并复制：

```
version: "3.9"
services:
  web: # web 服务
    build: .
    ports:
      - "5000:5000"
  redis: # redis 服务
    image: "redis:alpine"
```

`web` service 使用通过项目目录`Dockerfile`构建的镜像来创建容器。

`redis` service 则使用 Docker 官方仓库中的公共镜像。

### 4. 使用 Compose 构建和运行应用

1. 在项目目录中，执行`docker-compose up`启动应用。

   Compose 会拉取 redis 镜像，构建应用镜像，并启动定义的服务。

2. 在浏览器中输入`http://localhost:5000/`查看应用运行结果。你应该可以看到如下信息：

   ```
   Hello World! 该页面已被访问 1 times.
   ```

3. 停止应用有两种方式：

   - 在执行`docker-compose up`的终端下按下`CTRL+C`
   - 另起一个终端并切换到应用项目目录，执行`docker-compose down`

   两者的区别在于，`down`子命令“默认”会删除创建的容器和默认网络。

### 5. 编辑 Compose 文件添加 mount 绑定

编辑项目中的`docker-compose.yml`，为`web`服务添加 mount 绑定：

```
version: "3.9"
services:
  web:
    build: .
    ports:
      - "5000:5000"
    volumes:
      - .:/code # bind mount
    environment:
      FLASK_ENV: development
  redis:
    image: "redis:alpine"
```

新的`volumes`键将主机上的项目目录（当前目录）挂载到容器内的`/code`目录（容器的工作目录），这样允许我们动态修改代码，而不用重新构建镜像。`environment`键设置`FLASK_ENV`环境变量以告知`flask run`以开发模式运行，以便在代码发生变更时重载。

### 6. 重构并执行应用

在项目目录执行`docker-compose up	`即可。

### 7. 更新程序

现在，程序代码通过 volume 挂载到了容器内部，我们可以修改代码，并可以立即查看修改后的效果，不需要再重构镜像。

例如，修改代码如下：

```
return 'Hello from Docker! 该页面已被访问 {} 次.\n'.format(count)
```

刷新浏览器中的应用页面，更新应该已经呈现出来。

# 4. 子命令简介

对于 Compose 来说，大部分命令的对象既可以是项目本身，也可以指定为项目中的服务或者容器。如果没有特别的说明，命令对象将是项目，这意味着项目中所有的服务都会受到命令影响。

执行 `docker-compose [COMMAND] --help` 或者 `docker-compose help [COMMAND]` 可以查看具体某个命令的使用格式。

## 4.1 命令基本使用格式

`docker-compose`命令的基本使用格式为：

``` 
docker-compose [-f=<args>...] [--profile <name>...] [options] [COMMAND] [ARGS...]
```

其中：

- `-f`（或 `--file FILE`）：指定使用的 Compose 模板文件，默认为 docker-compose.yml，可以多次指定。
- `--profile`：指定活动配置，用于启动相关服务。

### 常用 options

- `-p, --project-name NAME `：指定项目名称。
- `--project-directory PATH`：指定项目目录。
- `--verbose`：输出更多调试信息。
- `--log-level LEVEL`：设置日志级别。
- `-v, --version`：打印版本并退出。

## 4.2 命令说明

### build 构建（重构）项目中的服务

服务一旦构建后，将会带上一个标记名，例如对于 web 项目中的一个 db 服务，可能是 web_db。

可以随时在项目目录下运行 `docker-compose build` 来重新构建服务。

### config 验证 Compose 文件

验证 Compose 文件格式是否正确，若正确则显示配置，若格式错误显示错误原因。

### down 停机并清理资源

停止容器，并移除由`up`命令创建的容器，网络，数据卷，镜像等。

默认情况下，以下资源被删除：

- 基于 Compose 文件中定义的服务创建的容器。
- Compose 文件`networks`字段定义的网络。
- 默认网络，如果有使用的话。

### events 容器事件流

为项目中的每个容器创建容器事件流。

### exec 在服务内执行命令

相当于`docker exec`。默认情况下会为命令分配一个伪终端（tty）。

### help 显示命令的帮助和使用说明

### images 列出创建容器所使用的镜像

### kill 向服务发送信号

默认信号是`SIGKILL`，可以通过`-s`标志指定信号。

```
docker-compose kill -s <SIGNAL>
```

### logs 显示服务日志

### pause 暂停服务的容器

### port 打印端口绑定的公共端口

### ps 列出项目中的容器

```
docker-compose ps [options] [SERVICE...]
```

不提供`SERVICE`的情况下，列出所有正在运行的关联容器。反之，列出正在运行的与指定服务关联的容器。

#### 常用 options

- `--services`   显示服务
- `--filter KEY=VAL`  通过属性过滤服务
- `-a, --all`  显示所有容器，包括停止的容器

### pull 拉取关联镜像

```
docker-compose pull [options] [SERVICE...]
```

不提供`SERVICE`的情况下，拉取项目关联的所有镜像。反之，拉取`SERVICE`关联的镜像。

### push 推送镜像到各自的镜像仓库

```
docker-compose push [options] [SERVICE...]
```

例如，`docker-compose.yml`内容如下：

```
version: '3'
services:
  service1:
    build: .
    image: localhost:5000/yourimage # goes to local registry
    
  service2:
    build: .
    image: your-dockerid/yourimage # goes to your repository on Docker Hub
```

### restart 重启容器

```
docker-compose restart [options] [SERVICE...]
```

重启所有停止的和正在运行的容器。

如果修改了`docker-compose.yml`或`Dockerfile`，更改不会通过`restart`立即反应出来，需要重新`up`。

### rm 删除停止的容器

```
docker-compose rm [options] [SERVICE...]
```

默认情况下，关联的匿名数据卷不会被删除。

#### 常用 options

- `-f, --force`  无需确认，直接删除
- `-s, --stop`  如果需要，在删除前停止容器
- `-v`  移除关联的任何匿名数据卷。

不提供 options 执行`rm`命令会同时移除由`docker-compose up`或`docker-compose run`创建的一次性容器。

### run 在指定服务上执行一个命令

```
docker-compose run [options] [-v VOLUME...] [-p PORT...] [-e KEY=VAL...] [-l KEY=VAL...]
	SERVICE [COMMAND] [ARGS...]
```

命令执行在一个通过指定服务创建的新容器里，好包括新的资源（比如数据卷）。

有两个重要的区别：

1. `run`执行的命令会覆盖服务配置中的定义。

   例如，如果`web`服务配置中以`bash`命令启动容器，那么`run web python app.py`会以`python app.py`覆盖之。

2. `run`不会创建任何在服务配置中定义的端口。

   这样做是为了避免端口冲突。如果确实需要创建服务端口，可以指定`--service-ports`标志。

如果`run`命令所要启动的服务配置了 links，`run`命令首先会检查链接的服务是否在线，并启动停机的服务。一旦所有链接的服务都在线，`run`命令才执行传递的命令。如果不需要启动链接的服务，使用`--no-deps`标志。

如果希望在执行后删除容器，而不应用服务配置的重启策略，使用`--rm`标志。

#### 常用 options

- `-d, --detach`  后台运行容器
- `--name NAME`  为容器命名
- `--entrypoint CMD`  覆盖镜像中的 ENTRYPOINT
- `-e KEY=VAL`  设置环境变量，可多次使用
- `-l, --label KEY=VAL`  添加或覆盖标签，可多次使用
- `-u, --user=""`  用指定用户运行
- `--no-deps`  不启动链接的服务
- `--rm`  执行完毕后删除容器。`-d`时忽略
- `-p, --publish=[]`  添加一个端口映射
- `--service-ports`  创建服务定义的端口映射
- `--use-aliases`  在网络中使用服务的网络别名
- `-v, --volume=[]`  绑定数据卷
- `-T`  禁用默认的伪终端分配
- `-w, --workdir=""`  指定容器内的工作目录

### start 启动已存在的容器

### stop 停止运行中的容器

```
docker-compose stop [options] [SERVICE...]
```

options 只有`-t, --timeout TIMEOUT`，指定停机超时时间，单位秒。

### top 显示服务中运行的进程信息

### unpause 回复暂停的容器

### up 为服务构建，创建，启动，附加容器，启动项目

```
docker-compose up [options] [--scale SERVICE=NUM...] [SERVICE...]
```

`up`命令用来启动整个项目。根本上，执行`up`命令会为项目中的服务构建，创建、启动，附加容器。

`up`同样会启动任何链接的服务，除非他们已经启动。

执行`docker-compose up`会聚合所有容器的输出，本质上就像执行`docker-compose up -d`后，再执行`docker-compose logs --follow`。

如果执行`docker-compose up`时，服务已存在关联的容器，并且服务的配置或者镜像已经发生变更，`up`会通过停止和再创建容器来采纳变更，过程中保留挂载的数据卷。可以使用`--no-recreate`标志防止这一行为。

执行`docker-compose up`时如果需要强制停止和再创建所有容器，使用`--force-recreate`标志。

#### 常用 options

- `-d, --detach`  后台执行，不兼容`--abort-on-container-exit`
- `--no-deps`  不启动链接的服务，即依赖服务
- `--force-recreate` 强制重新创建容器
- `--always-recreate-deps` 重新常见依赖容器，不兼容`--no-recreate`
- `--no-recreate` 不重建容器，不兼容`--force-recreate`、`--renew-anon-volumes`
- `--no-start` 创建后不启动服务
- `--build` 启动容器前构建镜像
- `--abort-on-container-exit` 一旦有容器停止，停止所有容器
- `--attach-dependencies` 附加依赖容器
- `-t, --timout TIMEOUT` 设置容器的停机超时秒数
- `-V, --renew-anon-volumes` 重建匿名数据卷而不从先前的容器中获取数据
- `--remove-orphans` 移除 Compose 文件中没有定义的容器和服务
- `--exit-code-from SERVICE` 返回指定服务的推出码
- `--scale SERVICE=NUM` 将制定服务扩展到 Num 个实例，覆盖 Compose 文件中配置的扩展设置 

# 5. Compose 文件参考

Compose 文件是使用 `Compose` 的核心，涉及到的指令关键字也比较多。但大家不用担心，这里面大部分指令跟 `docker run` 相关参数的含义都是类似的。

默认的 Compose 文件路径为 `./docker-compose.yml`，格式为 YAML 格式。用于定义 services（服务）, networks（网络） 和 volumes（数据卷）。

```
version: "3"

services:
  webapp:
    image: examples/web
    ports:
      - "80:80"
    volumes:
      - "/data"
```

如果使用 `build` 字段，在 `Dockerfile` 中设置的选项(例如：`CMD`, `EXPOSE`, `VOLUME`, `ENV` 等) 将会自动被获取，无需在 `docker-compose.yml` 中重复设置。

Compose 文件中可以使用类似于 bash 的`${VARIABLE}`语法在配置中使用环境变量。

## 5.1 service 配置参考

service 定义包含为该服务创建、启动容器时需要应用的所有参数，就像是给`docker run`命令传递命令行参数。

注意每个服务都必须通过 `image` 字段指定镜像或 `build` 字段（需要 Dockerfile）等来自动构建生成镜像。

与`docker run`一样，在 Dockerfile 中指定`CMD`，`EXPOSE`, `VOLUME`, `ENV`等选项默认情况下都被遵守，不需要在`docker-compose.yml`文件中指定。

### build 在构建时应用的配置选项

`build`可以指定为包含上下文路径的字符串：

```
version: "3.9"
services:
  webapp:
    build: ./dir
```

或者，作为一个 object，在`context`下指定路径，还有可选的`Dockerfile`和`args`参数：

```
version: "3.9"
services:
  webapp:
    build:
      context: ./dir
      dockerfile: Dockerfile-alternate
      args:
        buildno: 1
```

如果你既指定了`image`也指定了`build`，那么 Compose 用`webapp`和`image`中指定的可选标记来命名构建的镜像:

```
build: ./dir
image: webapp:tag
```

这将产生一个名为`webapp`的镜像和带有`tag`的标记，从`./dir`构建。

### cap_add, cap_drop 添加或删除容器功能

### cgroup_parent 为容器指定可选的父 cgroup

### command 覆盖容器的默认命令

```
command: bundle exec thin -p 3000

# 也可以像 Dockerfile 中的 CMD 的 exec 模式那样
command: ["bundle", "exec", "thin", "-p", "3000"]
```

### configs 授权服务配置的访问权

基于对每个服务配置`configs`来授予服务访问 configs 的权限。

> 配置必须已经存在或在配置文件的`top-level`的`configs`中定义。

configs 支持两种语法变体。

#### 短语法

短语法只需要指定配置名，这样就会授予容器配置的访问权，并且将配置挂载到容器内的`/<config_name>`。

```
version: "3.9"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 1
    configs:
      - my_config # ./my_config.txt 会被挂载到容器内的 /my_config
      - my_other_config
configs:
  my_config:
    file: ./my_config.txt
  my_other_config:
    external: true
```

#### 长语法

长语法关于在服务的任务容器中如何创建配置提供了更多的粒度。

- `source`： 在配置中定义的配置的标识符，如前面的 my_config
- `target`：挂载到服务容器内部的路径和名字。未指定的默认情况下为`/<source>`
- `uid`和`gid`：指定在服务容器内部挂载的配置的所有者。默认情况都为`0`。
- `mode`：容器内部挂载配置的访问权限，8进制。默认为`0444`。

```
version: "3.9"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 1
    configs:
      - source: my_config
        target: /redis_config # 将my_config挂载到容器内部的 /redis_config
        uid: '103'
        gid: '103'
        mode: 0440 # group-readable
configs:
  my_config:
    file: ./my_config.txt
  my_other_config:
    external: true
```

你可以为服务授权多个配置的访问权，并且可以混合使用长语法和短语法。

只定义配置并不会自动授予服务访问权。

### container_name 指定自定义的容器名

```
container_name: my-web-container
```

由于 Docker 中的容器名必须是唯一的，你就不能给一个指定了自定义名称的服务扩展超过 1 个容器。

### credential_spec 为受管理的服务账户配置证书规范（Windows）

### depends_on 服务之间依赖关系

服务之间的依赖导致如下行为：

- `docker-compose up`以依赖顺序启动服务。
- `docker-compose up SERVICE`自动包含服务的依赖，启动服务的依赖服务。
- `docker-compose stop`以依赖顺序停止服务。

一个简单的例子：

```
version: "3.9"
services:
  web:
    build: .
    depends_on: # 服务以 db、redis、web 的顺序启动，以 web 先于 db 和 redis 停止
      - db
      - redis
  redis:
    image: redis
  db:
    image: postgres
```

> 使用 depends_on 时有几件事需要注意：
>
> - `depends_on`在启动`web`之前，不会等待直到`db`和`redis`已准备就绪（状态），只会等到他们启动成功。
> - 当使用`verison 3`的 Compose 文件来以`swarm mode`部署堆栈时，`depends_on`会被忽略。

### deploy 指定与服务部署和运行相关的配置

只在用`docker stack deploy`部署`swarm`的情况下生效，会被`docker-compose up`和`docker-compose run`忽略。

```
version: "3.9"
services:
  redis:
    image: redis:alpine
    deploy:
      replicas: 6
      placement:
        max_replicas_per_node: 1
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
```

### devices 设备映射列表

与`--device`选项采用相同的格式。

### dns 自定义 DNS 服务器

### dns_search 自定义 DNS 查找域名

### entrypoint 覆盖默认的 entrypoint

### env_file 从文件中添加环境变量

### environment 定义环境变量

### expose 暴露内部端口

暴露的端口不会映射到主机端口，只能被内部链接的服务访问，只能指定`internal`端口。

### external_links 链接外部容器

链接到`docker-compose.yml`外，甚至是 Compose 外启动的容器（我的理解是，非 Compose 启动的容器），尤其是对于提供共享和公共服务的容器。当同时指定容器名和链接别名时（ CONTAINER:alias ），`external_links`遵循与`links`相同的语义。

```
external_links:
  - redis_1
  - project_db_1:mysql
  - project_db_1:postgresql
```

> 提醒：
>
> ​		外部创建的容器必须至少连接到一个与链接到他们的服务相同的网络。
>
> ​		推荐使用`networks`替代`links`。

### extra_hosts 添加主机名映射

添加主机名映射，使用与`--add-host`选项参数相同的值。

```
extra_hosts:
  - "somehost:162.242.195.82"
  - "otherhost:50.31.209.229"
```

包含主机名和IP地址的条目会在容器内的`/etc/hosts`添加：

```
$ cat /etc/hosts

162.242.195.82 somehost
50.31.209.229 otherhost
```

### healthcheck 健康检查

配置检查功能，用于在运行时检查服务容器是否处于健康状态。其功能与 Dockerfile 中的`HEALTHCHECK`指令一致。

```
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 1m30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

为了禁用镜像设置的默认检查，可以使用`disable:true`字段，相当于`test:["NONE"]`:

```
healthcheck:
  disable: true
```

### image 指定基于什么镜像创建人容器

如果同时指定了`build`，会用`image`指定的内容来标识镜像。

### init 容器内运行 init 进程

在容器内启动 init 进程来转发信号和获取进程。

```
version: "3.9"
services:
  web:
    image: alpine:latest
    init: true
```

### isolation 指定容器的隔离技术

### labels 为容器添加 metadata

可以使用数组或字典。 建议使用反向dns表示法，以防止你的标签与其他软件使用的标签发生冲突。

```
labels:
  com.example.description: "Accounting webapp"
  com.example.department: "Finance"
  com.example.label-with-empty-value: ""
```

```
labels:
  - "com.example.description=Accounting webapp"
  - "com.example.department=Finance"
  - "com.example.label-with-empty-value"
```

### links 连接到其它服务的容器（legacy，用[`networks`](###networks 需要加入的网络)替代）

可以同时指定服务名和链接别名（ "SERVICE:ALIAS" ），或者只指定服务名。

```
web:
  links:
    - "db"
    - "db:database"
    - "redis"
```

链接服务中的容器可以通过与别名相同的主机名访问，或者在没有指定别名的情况下通过服务名访问。

服务之间并不需要链接来进行通信，因为默认情况下，任何服务可以通过服务名访问任何其它服务。

`links`还与[`depends_on`](###depends_on 服务之间依赖关系)一样的方式来决定依赖顺序。

> 提醒：
>
> 如果同时指定`links`和`networks`，那些具有`links`的服务之间必须至少共享同一个相同的网络。

### logging 服务日志配置

```
logging:
  driver: syslog
  options:
    syslog-address: "tcp://192.168.0.42:123"
```

### network_mode 网络模式

使用与`--network`选项参数相同的值，加上特殊的形式`service:[service name]`。

```
network_mode: "bridge"
network_mode: "host"
network_mode: "none"
network_mode: "service:[service name]"
network_mode: "container:[container name/id]"
```

### networks 需要加入的网络

引用顶级`networks`字段下的条目。

```
services:
  some-service:
    networks:
     - some-network
     - other-network
```

#### aliases 可选的主机名

`aliases`是服务在网络内的别名。网络内的其它容器可以通过服务名或这个别名连接到服务的一个容器。

别名的作用域是网络，所有不同的网络中可以有相同的别名。

```
version: "3.9"

services:
  web:
    image: "nginx:alpine"
    networks:
      - new

  worker:
    image: "my-worker-image:latest"
    networks:
      - legacy

  db:
    image: mysql
    networks:
      new:
        aliases:
          - database
      legacy:
        aliases:
          - mysql

networks:
  new:
  legacy:
```

> 提醒：
>
> 网络范围内的别名可被多个容器，甚至多个服务共享。如果是这样，则不能保证别名确切解析道哪个容器。

#### ipv4_address, ipv6_address 网络内静态IP

指定容器在网络内的静态IP值。在顶级`networks`字段定义的网络配置中必须具备`ipam`块，其中包括子网的配置。

### pid 与主机操作系统共享 PID 地址空间

### ports 暴露端口

> 提醒：
>
> - 不与`network_mode: host`兼容。
> - `docker-compose run`忽略`ports`，除非`--service-ports`。

#### 短语法

三种选项:

- `HOST_PORT:CONTAINER_PORT`
- 只指定容器内部端口`CONTAINER_PORT`，会随机选择一个可用的主机端口
- `HOSTIPADDR:HOSRTPORT:CONTAINERPORT`或`HOSTIPADDR::CONTAINERPORT`，绑定端口到主机的指定IP

> 提醒：
>
> YAML 基于 60 来解析`XX:YY`格式的数字，如果使用低于 60 的容器端口会得到错误的结果。建议总是用字符串来指定端口映射值。

```
ports:
  - "3000"
  - "3000-3005"
  - "8000:8000"
  - "9090-9091:8080-8081"
  - "49100:22"
  - "127.0.0.1:8001:8001"
  - "127.0.0.1:5000-5010:5000-5010"
  - "127.0.0.1::5000"
  - "6060:6060/udp"
  - "12400-12500:1240"
```

#### 长语法

长语法形式可以配置如下字段：

- `target`: 容器内端口
- `published`: 映射到主机上的端口
- `protocol`: 协议(`tcp` or `udp`)
- `mode`: `host` for publishing a host port on each node, or `ingress` for a swarm mode port to be load balanced.

```
ports:
  - target: 80
    published: 8080
    protocol: tcp
    mode: host
```

### profiles 服务启用配置

```
profiles: ["frontend", "debug"]
profiles:
  - frontend
  - debug
```

`profiles`定义了一个是否启用该服务的命名配置列表。如果不配置，服务总是会启用。

### restart 重启策略

`no`是默认的重启策略，在任何情况下都不会重启容器。

`always`容器总是会重启。

`on-failure`当容器的退出码指示错误时重启。

`unless-stopped`除非容器停止，否则总是重启。

### secrets 配置`secrets`访问权

### security_opt 覆盖默认的标签方案

### stop_grace_period 指定容器停止前的等待时间

指定在容器如果未处理 `SIGTERM` 或`stop_signal`指定的信号的情况下，发送`SIGKILL`之前等待的时间。默认情况下会等待 10 秒。

### stop_signal 停止信号

指定可选的停止信号。默认使用`SIGTERM`。

### sysctls 容器中的内核参数

```
sysctls:
  net.core.somaxconn: 1024
  net.ipv4.tcp_syncookies: 0
  
sysctls:
  - net.core.somaxconn=1024
  - net.ipv4.tcp_syncookies=0
```

### tmpfs 容器中挂载临时文件系统

```
tmpfs: [/run]

tmpfs:
  - /run
  - /tmp
```

### ulimits 覆盖容器内的 limits

```
ulimits:
  nproc: 65535
  nofile:
    soft: 20000
    hard: 40000
```

### users_mode 为用户禁用 user namespace

### volumes 数据卷

将主机路径或命名的数据卷，指定为子选项挂载到容器内部。

当为单个服务绑定主机路径时，不需要定义顶级的`volumes`；如果是跨多个服务重用数据卷，则需要定义顶级的命名数据卷。

如果是引用命名数据卷，也需要定义顶级的`volumes`。

```
version: "3.9"
services:
  web:
    image: nginx:alpine
    volumes:
      - type: volume
        source: mydata # named volume
        target: /data
        volume:
          nocopy: true
      - type: bind
        source: ./static # host path
        target: /opt/app/static

  db:
    image: postgres:latest
    volumes:
      - "/var/run/postgres/postgres.sock:/var/run/postgres/postgres.sock" # host path
      - "dbdata:/var/lib/postgresql/data" # named volume

volumes:
  mydata:
  dbdata:
```

#### 短语法

短语法的格式为：

```
[SOURCE:]TARGET[:MODE]
```

其中：

- `SOURCE`：主机路径或卷名
- `TARGET`：挂载到容器内部的路径
- `MODE`：默认是`rw`（读写），还有`ro`（只读）

```
volumes:
  # Just specify a path and let the Engine create a volume
  - /var/lib/mysql

  # Specify an absolute path mapping
  - /opt/data:/var/lib/mysql

  # Path on the host, relative to the Compose file
  - ./cache:/tmp/cache

  # User-relative path
  - ~/configs:/etc/configs/:ro

  # Named volume
  - datavolume:/var/lib/mysql
```

#### 长语法

长语法支持额外的字段：

- `type`: 挂载类型， `volume`, `bind`, `tmpfs` or `npipe`
- `source`: 挂载源，主机路径或命名卷，命名卷必须定义在顶级的`volumes`中。`tmpfs`不支持。
- `target`: 容器内路径
- `read_only`: 是否只读
- `bind`: 额外的绑定选项
  - `propagation`: 用于绑定的传播模式
- `volume`: 额外的卷选项
  - `nocopy`: 创建卷时禁止从容器内复制数据
- `tmpfs`: 额外的临时文件系统选项
  - `size`: 零食文件系统的大小（字节）

```
version: "3.9"
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - type: volume
        source: mydata
        target: /data
        volume:
          nocopy: true
      - type: bind
        source: ./static
        target: /opt/app/static

networks:
  webnet:

volumes:
  mydata:
```

> 使用长语法时，需要事先创建被引用的文件夹

#### 数据卷之于 services, swarms, stack files

在没有为命名卷指定`source`的情况下，Docker 为支持服务的每个容器创建一个匿名卷。匿名卷在关联容器被移除后不会再保留。

如果想要持久保存数据，就使用一个命名卷和支持多主机的卷驱动程序，这样数据就能被任何节点访问到。或者，对服务设置约束，以便将其容器部署在存在卷的节点上。

### domainname, hostname, ipc, mac_address, privileged, read_only, shm_size, stdin_open, tty, user, working_dir

每个都对应单个值，类似于`docker run`运行时对应的值。

`mac_address`为 legacy。

### 指定持续时间（durations）

```
2.5s
10s
1m30s
2h32m
5h34m56s
```

### 指定字节大小

```
2b
1024kb
2048k
300m
1gb
```

## 5.2 volume 参考

顶级的`volumes`声明，可以允许你创建跨多服务使用的具名数据卷，并且可以使用 Docker 命令行进行检索和检视。

`volumes`下配置的条目可以为空，代表采用默认的 driver （多数情况下，为`local`）。

### driver 驱动

### driver_opts 驱动选项

### external 数据卷是否外部（Compose）创建

如果设置为`true`，即指出该数据卷已在 Compose 外部创建，执行`docker-compose up`不会试图创建它，并且如果数据卷不存在会报错。

### labels 为数据卷添加元数据

### name 自定义名称

自定义名称可用于引用包含特殊字符的数据卷。

## 5.3 network 参考

顶级的`networks`赋予你指定需要创建的网络的能力。

### driver 驱动

默认的网络驱动依赖于 Docker Engine 是如何被配置的，多数情况下，单主机默认为`bridge`，Swarm 上为`overlay`。

#### bridge 桥接 （single host）

#### overlay 覆盖 （swarm）

#### host or none （stack）

### driver_opts 驱动选项

### attachable （仅在`driver=overlay`时起作用）

### enable_ipv6 

### ipam IP地址管理

### internal

### labels

### external 外部网络

### name 自定义名称

## 5.4 configs 参考

顶级的`configs`声明定义或引用可以用于此`stack`中服务的配置。配置中的`source`可以是`file`或`external`。

- `file`: 配置是以指定路径下的文件内容来创建的
- `external`: 如果为`true`，指示配置已经存在
- `name`：自定义配置名
- `driver`和`driver_opts`:  自定义 secret driver 的名称和选项
- `template_driver`：templating driver 的名称，控制是否和怎样将 secret payload 作为模版进行批评估。

## 5.5 secrets 参考

顶级的`secrets`声明定义或引用`stack`中能够被服务访问的`secrets`。`secret`的`source`可以是`file`或`external`。

- `file`: `secret`有指定路径的文件内容创建
- `externel`：如果为`true`，指出`secret`已经创建
- `name`：自定义名称
- `template_driver`: 同[configs](##5.4 configs 参考)

## 5.6 变量替换

配置选项中可以包含环境变量，Compose 使用来自 shell environment 的变量值。

我们可以在 Compose 项目中的`.env`文件中定义环境变量的默认值，Compose 会自动去其中查找环境变量。在 shell environment 中的设置的值回覆盖`.env`文件中的值。

### 语法

`$VARIABLE`和`${VARIABLE}`都支持。

使用典型的 shell 语法，可以提供内连的默认值：

- `${VARIABLE:-default}` 如果`VARIABLE`未设置或者值为空
- `${VARIABLE-default}` 如果`VARIABLE`未设置

声明强制变量：

- `${VARIABLE:?err}` exit with err，当未设置或值为空
- `${VARIABLE:err}` exit with err, 当值未设置

## 5.7 Extension Fields 扩展字段

配置片段可以通过 extension fields 进行重用。这些特殊字段可以是任何格式，只要他们位于 Compose 文件的根目录下，并且他们的名字以`x-`打头。

```
version: "3.9"
x-custom: # extension field
  items:
    - a
    - b
  options:
    max-size: '12m'
  name: "custom"
```

扩展字段会被 Compose 忽略，但是可以通过 YAML 锚点插入到你的资源定义中。比如，你想要多个服务使用相同的 logging 配置：

```
logging:
  options:
    max-size: '12m'
    max-file: '5'
  driver: json-file
```

定义扩展字段，通过语法引用：

```
version: "3.9"
x-logging:
  &default-logging # 声明
  options:
    max-size: '12m'
    max-file: '5'
  driver: json-file

services:
  web:
    image: myapp/web:latest
    logging: *default-logging #引用
  db:
    image: mysql:latest
    logging: *default-logging #引用
```

# 6. 环境变量

Compose 有多个部分可以在某种意义上处理环境变量。

## 6.1 替代 Compose 文件中的环境变量

可以使用 shell 中环境变量的值来填充 Compose 文件中的值：

```
web:
  image: "webapp:${TAG}"
```

如果有多个环境变量，可以将他们添加到名为`.env`的默认的环境变量配置文件中，或者通过`--env-file`选项指定环境变量文件路径。

### `.env`文件

可以在`.env`文件中设置在 Compose 文件中引用到或用于配置 Compose 的任意环境变量的默认值。

`.env`文件的路径如下所示：

- 自`+v1.28`版开始，`.env`被放置在项目目录中。
- 项目目录可以通过`--file`选项或者`COMPOSE_FILE`环境变量明确指定，否则，项目目录为执行`docker-compose`命令的工作目录。

```
$ cat .env
TAG=v1.5

$ cat docker-compose.yml
version: '3'
services:
  web:
    image: "webapp:${TAG}"
```

当你执行`docker-compose up`命令，上面的`web`服务所使用的镜像会被替代为`webapp:v1.5`。你可以通过`config`命令来验证：

```
$ docker-compose config

version: '3'
services:
  web:
    image: 'webapp:v1.5'
```

在 shell 中定义的环境变量会优先于`.env`中定义的：

```
$ export TAG=v2.0
$ docker-compose config

version: '3'
services:
  web:
    image: 'webapp:v2.0'
```

可以通过`--env-file`选项指定`.env`文件路径。

### `--env-file`选项

指定的文件路径是相对于当前工作路径的，也就是执行`docker-compose`命令的路径。

```
$ cat .env
TAG=v1.5

$ cat ./config/.env.dev
TAG=v1.6


$ cat docker-compose.yml
version: '3'
services:
  web:
    image: "webapp:${TAG}"
```

在没有指定`--env-file`情况下，回默认读取`.env`：

```
$ docker-compose config 
version: '3'
services:
  web:
    image: 'webapp:v1.5'
```

指定`--env-file`会覆盖默认的文件路径：

```
$ docker-compose --env-file ./config/.env.dev config 
version: '3'
services:
  web:
    image: 'webapp:v1.6'
```

## 6.2 设置容器内环境变量

可以通过`environment`字段设置服务容器内的环境变量，就行执行`docker run -e VAR=VAL ...`:

```
web:
  environment:
    - DEBUG=1
```

## 6.3 传递环境变量到容器内

可以通过`environment`字段指定环境变量名而不指定值，从而将 shell 中对应的环境变量值传递到容器内，就像执行`docker run -e VAL ...`:

```
web:
  environment:
    - DEBUG
```

## 6.4 “env_file” 配置选项

可以通过在服务里配置`env_file`选项来向服务容器传递环境变量，就像执行`--env-file=FILE ...`:

```
web:
  env_file:
    - web-variables.env
```

## 6.5 为`docker-compose run`设置环境变量

可以通过`docker-compose run -e`为一次性容器设置环境变量：

```
docker-compose run -e DEBUG=1 web python console.py

docker-compose run -e DEBUG web python console.py
```

当在多个文件中设置相同的环境变量时，Compose 选择变量值的优先级位：

1. Compose 文件
2. shell 环境变量
3. 环境变量文件
4. Dockerfile
5. 未定义变量

下面的例子中，我们同时在环境变量文件和 Compose 文件中设置变量：

```
$ cat ./Docker/api/api.env
NODE_ENV=test

$ cat docker-compose.yml
version: '3'
services:
  api:
    image: 'node:6-alpine'
    env_file:
     - ./Docker/api/api.env
    environment:
     - NODE_ENV=production
```

容器运行时，Compose 文件中的环境变量优先采用：

```
$ docker-compose exec api node

> process.env.NODE_ENV
'production'
```

## 6.6 用环境变量配置 Compose

有若干可用于配置 Docker Compose 命令行行为的环境变量，他们以`COMPOSE_`或`DOCKER_`为前缀。

官方文档：[CLI Environment Variables](https://docs.docker.com/compose/reference/envvars/)

## 6.7 环境变量文件参考

Compose 在项目目录下的`.env`文件中声明环境变量的默认值。

项目目录的采纳优先级如下：

- `--project-directory`指定的目录
- 第一个`--file`所在目录
- 当前工作目录

### 语法规则

- 每行格式为`VAR=VAL`
- 以`#`打头的行会当作注释对待
- 空白行会被忽略
- 引号会作为变量值的一部分，不会做特殊处理

### Compose 文件与CLI 变量

在环境变量文件中定义的换奖变量主要用于 Compose 文件中的变量替换，也可以在其中指定 CLI 变量：

- `COMPOSE_API_VERSION`
- `COMPOSE_CONVERT_WINDOWS_PATHS`
- `COMPOSE_FILE`
- `COMPOSE_HTTP_TIMEOUT`
- `COMPOSE_PROFILES`
- `COMPOSE_PROJECT_NAME`
- `COMPOSE_TLS_VERSION`
- `DOCKER_CERT_PATH`
- `DOCKER_HOST`
- `DOCKER_TLS_VERIFY`

> 提醒：
>
> - 运行时指定的变量值总是会覆盖`.env`文件中的值。例如，由命令行参数`-e`传入的值会优先采纳。
> - 定义在`.env`文件中的环境变量在容器内不是自动可见的。要设置适用于容器的环境变量，请遵循[ 6. 环境变量](# 6. 环境变量)章节内容前面部分提到的方法。

# 7. 使用 profiles 禁/启用服务

profiles 允许通过可选择的启用服务来根据不同的使用环境调整 Compose 程序模型。这是通过给服务分配 0 或多个 profiles 来实现的。如果未分配，服务将默认总是启用；否则，只有在 profiles 激活时才启用。

这样就允许在同一个`docker-compose.yml`文件中定义额外的服务来满足特定的场景，比如调试和开发。

## 7.1 为服务分配 profiles

```
version: "3.9"
services:
  frontend:
    image: frontend
    profiles: ["frontend"]

  phpmyadmin:
    image: phpmyadmin
    depends_on:
      - db
    profiles:
      - debug

  backend:
    image: backend

  db:
    image: mysql
```

执行`docker-compose up`的话，只会启动`backend`和`db`服务。

> 核心服务不应该添加 profiles，应该总是启用并自动启动

## 7.2 启用 profiles

通过指定`--profile`命令行选项或设置`COMPOSE_PROFILES`环境变量来启用 profile：

```
$ docker-compose --profile debug up
$ COMPOSE_PROFILES=debug docker-compose up
```

对前面的 Compose 文件执行以上命令将会启动`backend`, `db`和`phpmyadmin`服务。

profile 可以指定多个：

```
$ docker-compose --profile frontend --profile debug up
$ COMPOSE_PROFILES=frontend,debug docker-compose up
```

## 7.3 自动启用 profiles 与 依赖性解析

如果命令行明确指定启动某个分配了 profiles 的服务，则不需要手动提供`--profile`来启动。可用于一次性服务和调试工具。

```
version: "3.9"
services:
  backend:
    image: backend

  db:
    image: mysql

  db-migrations:
    image: backend
    command: myapp migrate
    depends_on:
      - db
    profiles:
      - tools
```

```
# will only start backend and db
$ docker-compose up -d

# this will run db-migrations (and - if necessary - start db)
# by implicitly enabling profile `tools`
$ docker-compose run db-migrations
```

注意，`docker-compose`只会自动启用命令行指定服务的 profiles，不会启用依赖项的 profiles。这意味着，所有目标服务依赖的服务，必须具备一个公共的 profile，或者显示指定一个匹配的 profile。

```
version: "3.9"
services:
  web:
    image: web

  mock-backend:
    image: backend
    profiles: ["dev"]
    depends_on:
      - db

  db:
    image: mysql
    profiles: ["dev"]

  phpmyadmin:
    image: phpmyadmin
    profiles: ["debug"]
    depends_on:
      - db
# will only start "web"
$ docker-compose up -d

# this will start mock-backend (and - if necessary - db)
# by implicitly enabling profile `dev`
$ docker-compose up -d mock-backend

# this will fail because profile "dev" is disabled
$ docker-compose up phpmyadmin
```

虽然指定`phpmyadmin`服务会自动启用 profiles - `debug`，但不会自动启用`db`的 profiles，例如`dev`。可以通过给`db`添加`debug` profile，或者明确指定一个`db`的 profile 来修复这个问题：

```
db:
  image: mysql
  profiles: ["debug", "dev"]
```

```
# profile "debug" is enabled automatically by targeting phpmyadmin
$ docker-compose --profile dev up phpmyadmin
$ COMPOSE_PROFILES=dev docker-compose up phpmyadmin
```

# 8. Compose 中的网络

默认情况下，Compose 会设置一个默认网络。每一个 Compose 创建的容器都会加入到默认网络中，并且可以被网络中的其它容器访问，以与容器名相同的主机名来被发现。

> 注意：
>
> 网络名基于项目名创建

举个例子，假设你的 APP 所在目录未`myapp`，并且 Compose 文件内容如下：

```
version: "3.9"
services:
  web:
    build: .
    ports:
      - "8000:8000"
  db:
    image: postgres
    ports:
      - "8001:5432"
```

执行`docker-compose up`：

1. 网络`myapp_default`创建
2. 基于`web`配置创建容器，以名称`web`加入`myapp_default`
3. 基于`db`配置创建容器，以名称`db`加入`myapp_default`

现在，容器可以被通过主机名`web`或`db`查找到，并返回合适的 IP。

了解`HOST_PORT`和`CONTAINER_PORT`之间的区别尤其重要。服务之间通过`CONTAINER_PORT`沟通，而`HOST_PORT`用于对外提供访问。对于`web`容器，通过`postgres://db:5432`连接到`db`；而对于主机端，比如主机上的其它应用需要访问`db`，则通过`pstgres://{DOCKER_IP}:8001`，其中`DOCKER_IP`为启动 Docker 环境的主机IP。

## 8.1 更新容器

更新服务配置后，需要执行`docker-compose up`来更新正在运行的容器，旧的容器将被移除，新创建的容器会以不同的 IP 相同的主机名加入网络中。运行中的容器可以通过相同的名称查找新主机的 IP。

如果有容器连接到了旧的容器上，连接将会被关闭。检测这种情况并恢复连接是容器自己的职责。

## 8.2 链接

links 允许为可以访问的服务定义额外的别名。默认情况下，任何 service 通过 service name 可以被其它任何 service 访问。下面的例子中，`db`通过主机名`db`或`database`可以被访问：

```
version: "3.9"
services:

  web:
    build: .
    links:
      - "db:database"
  db:
    image: postgres
```

## 8.3 多主机网络

当在启用 Docker Swarm Mode 的情况下部署 Compose 应用，你可以使用内置的 overlay 驱动来启用多主机网络。

[Swarm Mode]: https://docs.docker.com/engine/swarm/
[Getting started with multi-host networking]: https://docs.docker.com/network/network-tutorial-overlay/

## 8.4 指定自定义网络

你可以在 Compose 文件中指定顶级的`networks`字段来声明自定义网络，用于替代默认网络。这允许你创建更复杂的网络拓扑，指定自定义网络驱动和选项。你还可以使用它来连接到不是由 Compose 管理的外部网络。

每一个服务都可以指定自己需要连接的网络，通过指定 service-level 的`networks`字段，它是一个用来引用在顶级`networks`字段下声明的网络的列表。

下面是一个定义两个自定义网络的Compose文件示例。`proxy`服务与`db`服务是隔离的，因为它们并不共享一个共同的网络——只有应用程序可以与两者通信。

```
version: "3.9"

services:
  proxy:
    build: ./proxy
    networks:
      - frontend
  app:
    build: ./app
    networks:
      - frontend
      - backend
  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
    # Use a custom driver
    driver: custom-driver-1
  backend:
    # Use a custom driver which takes special options
    driver: custom-driver-2
    driver_opts:
    foo: "1"
    bar: "2"
```

网络可以通过设置每个附加网络的`ipv4_address`和/或`ipv6_address`来配置静态IP地址（service_level）。

还可以给网络赋予一个名称：

```
version: "3.9"
services:
  # ...
networks:
  frontend:
    name: custom_frontend
    driver: custom-driver-1
```

## 8.5 配置默认网络

除了指定你自己的网络之外，你还可以通过在`networks`下定义一个名为default的条目来更改全应用范围的默认网络的设置:

```
version: "3.9"
services:
  web:
    build: .
    ports:
      - "8000:8000"
 db:
   image: postgres

networks:
  default:
    # Use a custom driver
    driver: custom-driver-1
```

## 8.6 使用已存在的网络

如果你想要你的容器加入一个已存在的网络，可以使用`external`选项：

```
services:
  # ...
networks:
  default:
    external: true
    name: my-pre-existing-network
```

上面的例子里，Compose 不会创建一个名为`[projectname]_default`的默认网络，而是查找名为`my-pre-existing-network`的网络并把它当作默认网络，然后使应用中的所有容器连接到它。

# 9. 控制启动顺序

使用[`depends_on`](###depends_on 服务之间依赖关系)选项可以控制服务的启动和关闭顺序。Compose 总是以依赖顺序来启动和停止容器，依赖顺序由`depends_on`, `links`, `volumes_from`, `network_mode: "service:..."`这些选项来决定。

然而，在启动时，Compose 并不会等待容器“ready”，只需要容器运行成功即可。那么，在某些情况下，就需要我们做一些额外操作，来保证启动过程不会失败。

例如，等待数据库就绪的问题，这实际上是分布式系统问题中的一个子集。在生产环境中，你的数据库可能随时变得不可用或者移机。你的应用程序需要一定的弹性来应对这样的问题。

为了得到弹性，将你的应用程序设计为，一旦遇到故障，就尝试重新建立数据库连接。只要重新连接，应用程序最终都能连接到数据库。

最好的方法是在程序内设计弹性。但是，如果你不需要这种弹性，那可以使用包装器脚本（wrapper script）来实现：

- 使用诸如 [wait-for-it](), [dockerize](), sh 兼容的 [wait-for](), 或者 [RelayAndContainers]() 模版等工具。这些小包装脚本可以包含在应用程序的镜像中，用于轮询给定的主机和端口，直到它接受TCP连接。

  比如，使用`wait-for-it.sh`或`wait-for`包装你的服务指令：

  ```
  version: "2"
  services:
    web:
      build: .
      ports:
        - "80:8000"
      depends_on:
        - "db"
      command: ["./wait-for-it.sh", "db:5432", "--", "python", "app.py"]
    db:
      image: postgres
  ```

  > 提示：
  >
  > 该解决方案有其局限性。例如，其并没有验证服务是否已经“就绪”。
  >
  > 如果你需要为`command`添加更多的参数，可在循环使用`bash shift`命令，就像后面例子中提及的。

- 或者，编写自己的包装器脚本来执行更特定于应用程序的运行状况检查。例如，您可能希望等待，直到Postgres准备好接受命令：

  ```
  #!/bin/sh
  # wait-for-postgres.sh
  
  set -e
    
  host="$1"
  shift # 相当于 shift “1”，将位于 1+1=2 位置的参数移动到 $1位置，$2 后面直到 $# 的参数也一次向左移动
  
  # 使用 psql 命令可以保证数据库已准备就绪
  until PGPASSWORD=$POSTGRES_PASSWORD psql -h "$host" -U "postgres" -c '\q'; do
    >&2 echo "Postgres is unavailable - sleeping"
    sleep 1
  done
    
  >&2 echo "Postgres is up - executing command"
  exec "$@"
  ```

  ```
  command: ["./wait-for-postgres.sh", "db", "python", "app.py"]
  ```


# 10. 在文件和项目之间共享 Compose 配置

Compose 支持两种共享公共配置的方法:

1. 通过使用多个 Compose 文件扩展整个 Compose 文件
2. 使用`extend`字段扩展单个服务（<=v2.1）

## 10.1 多 Compose 文件

### 10.1.1 理解多 Compose 文件

默认情况下，Compose 读取`docker-compose.yml`文件和一个可选的`docker-compose.overide.yml`文件。按照惯例，`docker-compose.yml`包含基本配置。而`override`文件，顾名思义，包含已存在服务或全新服务的配置覆盖。

如果服务在两个文件中都有定义，Compose 使用[添加和覆盖配置](#10.3 添加和覆盖配置)中描述的规则合并配置。

若要使用多个 override 文件，或使用不同名称的覆盖文件，可以使用`-f`选项指定文件列表。Compose 按命令行指定的顺序合并文件。

当使用多个配置文件时，必须确保所有的文件路径都是相对基本 Compose 文件的，即使用`-f`指定的第一个 Compose 文件。因为 override 文件不需要是有效的 Compose 文件，可以是包含配置的小片段。

### 10.1.2 示例

#### 不同环境

多 Compose 文件的一个通用例子是将开发环境转变为生产环境。为了支持这些差异，你可以将 Compose 配置分成几个不同的文件：

定义服务规范配置的基本文件`docker-compose.yml`：

```
web:
  image: example/my_web_app:latest
  depends_on:
    - db
    - cache

db:
  image: postgres:latest

cache:
  image: redis:latest
```

在本例中，开发配置向主机公开一些端口，将我们的代码挂载为一个卷，并构建web映像。

覆盖配置`docker-compose.override.yml`：

```
web:
  build: .
  volumes:
    - '.:/code'
  ports:
    - 8883:80
  environment:
    DEBUG: 'true'

db:
  command: '-d'
  ports:
    - 5432:5432

cache:
  ports:
    - 6379:6379
```

当运行`docker-compose run`时，它会自动读取覆盖文件。

现在，最好也在生产环境中使用 Compose 应用。为此，我们新建一个用于生产环境的覆盖文件`docker-compose.prod.yml`:

```
web:
  ports:
    - 80:80
  environment:
    PRODUCTION: 'true'

cache:
  environment:
    TTL: '500'
```

部署应用可以执行:

```
 docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

#### 管理性任务

另一个常见的用例是对 Compose 应用程序中的一个或多个服务运行临时或管理任务。这个示例演示了运行数据库备份。

基础 Compose 文件 **docker-compose.yml**：

```
web:
  image: example/my_web_app:latest
  depends_on:
    - db

db:
  image: postgres:latest
```

在 **docker-compose.admin.yml** 文件中定义用于数据库导出或备份的新服务：

```
dbadmin:
      build: database_admin/
      depends_on:
        - db
```

执行`docker-compose up -d`启动常规环境。运行数据库备份，需要包含 **docker-compose.admin.yml**文件：

```
 docker-compose -f docker-compose.yml -f docker-compose.admin.yml \
  run dbadmin db-backup
```

## 10.2 扩展服务

> 注意：
>
> `extends`字段在早期 Compose 文件格式到 v2.1 版都有支持，但在 Compose v3.x 中不支持。

Docker Compose 的`extends`关键字允许在不同的文件，甚至完全不同的项目之间共享公共配置。如果有多个服务重用一组公共配置选项，那么扩展服务非常有用。使用扩展，您可以在一个地方定义一组公共服务选项，并从任何地方引用它。

记住，`volumes_from`和`depends_on`永远不会在使用`extends`的服务之间共享。这些异常的存在是为了避免隐式依赖关系;你总是在本地定义`volumes_from`。这确保了在读取当前文件时，服务之间的依赖关系是清晰可见的。在本地定义这些也可以确保对引用文件的更改不会破坏任何东西。

### 10.2.1 理解 extends 配置

当在`docker-compose.yml`中定义任何服务时，你可以像这样声明你正在扩展其它服务：

```
services:
  web:
    extends:
      file: common-services.yml
      service: webapp
```

这指示 Compose 重用`common-service.yml`中定义的 webapp 服务的配置。假设`common-services.yml`是这样的:

```
services:
  webapp:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - "/data"
```

在这种情况下，您将得到在`docker-compose.yml`中的`web`服务下定义`build`，`ports`和`volumes`完全相同的结果。

你可以进一步在`docker-compose.yml`中定义或重新定义配置:

```
services:
  web:
    extends:
      file: common-services.yml
      service: webapp
    environment:
      - DEBUG=1
    cpu_shares: 5

  important_web:
    extends: web
    cpu_shares: 10
```

您还可以编写其他服务，并将您的 web 服务链接到它们:

```
ervices:
  web:
    extends:
      file: common-services.yml
      service: webapp
    environment:
      - DEBUG=1
    cpu_shares: 5
    depends_on:
      - db
  db:
    image: postgres
```

### 10.2.2 用例

当您有多个具有公共配置的服务时，扩展单个服务非常有用。下面的例子是一个 Compose 应用程序，它有两个服务: 一个 web 应用程序和一个 queue worker。这两个服务使用相同的代码库并共享许多配置选项。

在一个`common.yml`中我们定义了公共配置:

```
services:
  app:
    build: .
    environment:
      CONFIG_FILE_PATH: /code/config
      API_KEY: xxxyyy
    cpu_shares: 5
```

`docker-compose.yml` 中，我们定义了使用公共配置的具体服务:

```
services:
  webapp:
    extends:
      file: common.yml
      service: app
    command: /code/run_web_app
    ports:
      - 8080:8080
    depends_on:
      - queue
      - db

  queue_worker:
    extends:
      file: common.yml
      service: app
    command: /code/run_worker
    depends_on:
      - queue
```

## 10.3 添加和覆盖配置

Compose 将配置从原始服务复制到本地服务。如果在原有服务和本地服务中都定义了配置选项，则本地值将替换或扩展原有值。

### 单值选项

对于像`image`、`command`或`mem_limit`这样的单值选项，新值将替换旧值。

原始服务：

```
services:
  myservice:
    # ...
    command: python app.py
```

本地服务:

```
services:
  myservice:
    # ...
    command: python otherapp.py
```

结果：

```
services:
  myservice:
    # ...
    command: python otherapp.py
```

### 多值选项

对于多值选项`ports`、`expose`、`external_links`、`dns`、`dns_search`和`tmpfs`, Compose 连接这两组值:

原始服务：

```
services:
  myservice:
    # ...
    expose:
      - "3000"
```

本地服务：

```
services:
  myservice:
    # ...
    expose:
      - "4000"
      - "5000"
```

结果：

```
services:
  myservice:
    # ...
    expose:
      - "3000"
      - "4000"
      - "5000"
```

### 对于`environment`、`labels`、`volumes`和`devices`

对于`environment`、`labels`、`volumes`和`devices`，Compose 优先使用本地定义的值“合并”条目。对于`environment`和`labels`，环境变量或标签名决定使用哪个值:

原始服务：

```
services:
  myservice:
    # ...
    environment:
      - FOO=original
      - BAR=original
```

本地服务：

```
services:
  myservice:
    # ...
    environment:
      - BAR=local
      - BAZ=local
```

结果：

```
services:
  myservice:
    # ...
    environment:
      - FOO=original
      - BAR=local
      - BAZ=local
```

`volumes`和`devices`的条目使用容器中的挂载路径合并:

原始服务:

```
services:
  myservice:
    # ...
    volumes:
      - ./original:/foo
      - ./original:/bar
```

本地服务：

```
services:
  myservice:
    # ...
    volumes:
      - ./local:/bar
      - ./local:/baz
```

结果：

```
services:
  myservice:
    # ...
    volumes:
      - ./original:/foo
      - ./local:/bar
      - ./local:/baz
```

# 11. 生产环境

在开发中使用 Compose 定义应用程序时，可以使用此定义在不同的环境(如 CI、staging 和 production)中运行应用程序。

部署应用程序的最简单方法是在单个服务器上运行它，类似于运行开发环境的方式。如果您想扩展应用程序，可以在 Swarm 集群上运行 Compose 应用程序。

## 11.1 为生产环境修改 Compose 文件

您可能需要对应用程序配置进行更改，以使其准备好投入生产。这些变化可能包括:

- 删除应用程序代码的任何卷绑定，以便代码留在容器内，不能从外部更改
- 绑定主机的不同端口
- 以不同的方式设置环境变量，比如减少日志记录的冗长，或者为外部服务(如电子邮件服务器)指定设置
- 指定重新启动策略，避免停机，如`restart:always`
- 增加额外的服务，如日志聚合器

出于这个原因，考虑定义一个额外的 Compose 文件，比如`production.yml`，它指定适合生产环境的配置。这个配置文件只需要包括您想从原始 Compose 文件中进行的更改。附加的 Compose 文件可以应用于原始`docker-compose.yml`以创建新的配置。

获得第二个配置文件后，通过`-f`选项告诉Compose使用文件:

```
 docker-compose -f docker-compose.yml -f production.yml up -d
```

## 11.2 部署变更

当你改变你的应用程序代码时，记得重建你的映像和重新创建你的应用程序的容器。例如，要重新部署 web         服务，请使用:

```
docker-compose build web
docker-compose up --no-deps -d web
```

它首先为 web 重建图像，然后停止、破坏和重新创建 web 服务。`--no-deps`标志阻止 Compose 重新创建 web 所依赖的任何服务。

## 11.3 在单个服务器上运行 Compose

通过设置`DOCKER_HOST`、`DOCKER_TLS_VERIFY`和`DOCKER_CERT_PATH`环境变量，可以使用 Compose 将应用程序部署到远程 Docker 主机。

一旦设置了环境变量，所有普通的`docker-compose` 命令都可以工作，无需进一步配置。

# FAQ

## `up`,`run`和`start`子命令之间的区别

典型的，希望通过`up`启动或重启 Compose 文件中的所有服务。"attached"模式下，你将在控制台看到所有容器的日志输出。"detached"模式下，Compose 在启动容器后退出，容器们在后台运行。

`docker-compose run`用于运行一次性或特别的任务。需要指定你想要运行的服务名，并且只会启动该服务所依赖的服务的容器。使用`run`来运行测试或执行管理任务，例如移除或添加数据到数据卷容器。`run`命令的作用类似于`docker run -ti`，为容器开启一个交互式的终端，返回与容器中进程退出状态匹配的退出状态。

`start`命令仅仅用于重启先前创建的，并停止的容器。
