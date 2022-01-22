在比较复杂的项目中，某些子模块的迭代，不用过多的关注，只是引用其功能，这样的情况下，可以使用 git submodule 模式，子模块就可以作为独立的仓库管理，不用合并到主仓库中，或者在主仓库中拷贝一个子仓库的特定版本。

```shell
git submodule ...
```

# 创建

```shell
git submodule add <submodule_url>
```

创建子模块后，项目目录会多出一个`.gitmodules`文件，记录子模块路径、url、分支等信息。

# 拉取

子模块的创建者在添加子模块时，git 会自动 clone 子模块，而该仓库的后续使用者在 clone的时候，并不会自动拉取子模块。clone 者可以使用下面两种方法来拉取子模块：

- clone 时添加`--recurse-submodules`参数

- clone 后，在仓库目录下依次执行`git submodule init`，`git submodule update`

# 子模块有更新

1）当前项目下子模块文件夹内的内容发生了未跟踪的内容变动；

2）当前项目下子模块文件夹内的内容发生了版本变化；

3）当前项目下子模块文件夹内的内容没变，远程有更新；
