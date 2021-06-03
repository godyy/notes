【编者的话】这是关于Linux容器介绍的第一篇，主要介绍了Linux控制组：control groups，也叫做CGroups，以及进程隔离。通过一个简单的例子让你很快学习到Linux控制组是如何工作的。以及哪些库可以让你方便快捷的使用控制组。

*每个人都听说过容器，那么容器到底是什么呢？*

软件的发展使这项技术以多种方式得以实现，而Docker则是最流行的一种。因为容器的可移植性以及它隔离工作环境的特点可以限制它对底层计算的影响以及影响范围，越来越多数据中心开始采用这项技术。为了全面了解这项技术，你首先需要了解是哪些技术实现了容器。

*附注：人们总喜欢问容器和虚拟机的区别。它们都有各自特定的目的，并没有太多相似的地方。并且一个并不会淘汰掉另一个。一个容器指的是一个轻量级的环境，在这个环境中你可以启动一个或者多个应用，它们都与外界隔离，并且这个环境的性能与裸机相当。但如果你需要运行一整个操作系统或者生态系统，又或者你需要运行与底层环境不兼容的应用程序，那么你需要选择虚拟机。*

### Linux 控制组

说实话，一些未知的软件应用可能需要被控制或限制——至少是为了稳定性或者某种程度上的安全性。很多时候，一个bug或者仅仅只是烂代码就有可能破坏掉整个机器甚至可能削弱整个生态。幸运的是，有一种方式可以控制相同的应用程序，Linux控制组（cgroups）是一个内核功能，用于限制，记录和隔离一个或多个进程对CPU，内存，磁盘I/O，以及网络的使用量及访问。

控制组技术最初是由谷歌开发的，最终在2.6.24版本（2008年1月）中并入Linux内核主线。这项技术被部分重新设计，添加了kernfs（用于分割一些sysfs逻辑），这些改变被合并到3.15和3.16版本的内核中。

控制组主要为了提供统一接口来管理进程或者整个操作系统级别的虚拟化，包括Linux 容器，或者LXC（将在之后的文章中详细介绍这项技术）。控制组框架提供了以下功能：

- **资源限制**：一个控制组可以配置成不能超过指定的内存限制或是不能使用超过一定数量的处理器或限制使用特定的外围设备。
- **优先级**：一个或者多个控制组可以配置成使用更少或者更多的CPU或者磁盘I/O吞吐量。
- **记录**：一个控制组的资源使用情况会被监督以及测量。
- **控制**：进程组可以被冻结，暂停或者重启。

一个控制组可以包含一个或者多个进程，这些进程将全部绑定于同一组限制。控制组也可以继承，这意味着一个子组可以继承其父组限制。

Linux内核为控制组技术的一系列控制器以及子系统提供了访问。控制器将负责将特定类型的系统资源分配给一个控制组（包含一个或者多个进程）。例如，`内存`控制器会限制内存使用而`cpuacct`控制器会监控CPU的使用情况。

你可以直接或者间接（通过LXC，libvirt或者Docker）访问及管理控制组，这里我首先介绍使用sysfs以及`libgroups`库。接下来的示例需要你预先安装一个必须的包。在Red Hat Enterprise Linux或者CentOS里面，在命令行输入以下命令：

```
$ sudo yum install libcgroup libcgroup-tools
```


如果是Ubuntu或者Debian，输入：

```
$ sudo apt-get install libcgroup1 cgroup-tools
```


我将使用一个简单的shell脚本文件test.sh作为示例应用程序，它将会在无限`while`循环中运行以下两个命令。

```
$ cat test.sh
!/bin/shwhile [ 1 ]; do
    echo "hello world"
    sleep 60
done
```

### 手动方法

安装必要的包后，你可以直接通过sysfs的目录结构来配置你的控制组，例如，要在内存子系统中创建一个叫做`foo`的控制组，只需要在`/sys/fs/cgroup/memory`底下新建一个叫做`foo`的目录：

```
$ sudo mkdir /sys/fs/cgroup/memory/foo
```

默认情况下，每个新建的控制组将会继承对系统整个内存池的访问权限。但对于某些应用程序，这些程序拒绝释放已分配的内存并继续分配更多内存，这种默认继承方式显然不是个好主意。要使程序的内存限制变得更为合理，你需要更新文件`memory.limit_in_bytes`。

限制控制组`foo`下运行的任何应用的内存上限为50MB：

```
$ echo 50000000 | sudo tee
↪/sys/fs/cgroup/memory/foo/memory.limit_in_bytes
```

验证设置：

```
$ sudo cat memory.limit_in_bytes
50003968
```


请注意，回读的值始终是内核页面大小的倍数（即4096字节或4KB）。这个值是**内存的最小可分配大小**。

启动应用程序test.sh：

```
$ sh ~/test.sh 
```


使用进程ID（PID），将应用程序移动到`内存`控制器底下的控制组`foo`：

```
$ echo 2845 > /sys/fs/cgroup/memory/foo/cgroup.procs
```


使用相同的PID，列出正在运行的进程并验证它是否在正确的控制组下运行：

```
$ ps -o cgroup 2845
CGROUP
8:memory:/foo,1:name=systemd:/user.slice/user-0.slice/
↪session-4.scope
```

你还可以通过读取文件来监控控制组正在使用的资源。在这种情况下，你可以查看你的进程（以及生成的子进程）被分配的内存大小。

```
$ cat /sys/fs/cgroup/memory/foo/memory.usage_in_bytes
253952
```

### 当进程“迷路”时

现在让我们重新创建相同的场景，但这次我们将控制组`foo`的内存限制从50MB改为500 bytes：

```
$ echo 500 | sudo tee /sys/fs/cgroup/memory/foo/
↪memory.limit_in_bytes
```

前例中创建的控制组`foo`可用`rmdir`命令删除。

*注意：如果任务超出其定义的限制，内核将进行干预，并在某些情况下终止该任务*。

同样，当您重新读取值时，它将始终是内核页面大小的倍数。因此，虽然您将其设置为500字节，但它实际上被设置为4 KB：

```
$ cat /sys/fs/cgroup/memory/foo/memory.limit_in_bytes
4096
```

启动应用程序test.sh，将其移动到控制组下并监视系统日志：

```
$ sudo tail -f /var/log/messages
Oct 14 10:22:40 localhost kernel: sh invoked oom-killer:
↪gfp_mask=0xd0, order=0, oom_score_adj=0
Oct 14 10:22:40 localhost kernel: sh cpuset=/ mems_allowed=0
Oct 14 10:22:40 localhost kernel: CPU: 0 PID: 2687 Comm:
↪sh Tainted: G
OE  ------------   3.10.0-327.36.3.el7.x86_64 #1
Oct 14 10:22:40 localhost kernel: Hardware name: innotek GmbH
VirtualBox/VirtualBox, BIOS VirtualBox 12/01/2006
Oct 14 10:22:40 localhost kernel: ffff880036ea5c00
↪0000000093314010 ffff88000002bcd0 ffffffff81636431
Oct 14 10:22:40 localhost kernel: ffff88000002bd60
↪ffffffff816313cc 01018800000000d0 ffff88000002bd68
Oct 14 10:22:40 localhost kernel: ffffffffbc35e040
↪fffeefff00000000 0000000000000001 ffff880036ea6103
Oct 14 10:22:40 localhost kernel: Call Trace:
Oct 14 10:22:40 localhost kernel: [<ffffffff81636431>]
↪dump_stack+0x19/0x1b
Oct 14 10:22:40 localhost kernel: [<ffffffff816313cc>]
↪dump_header+0x8e/0x214
Oct 14 10:22:40 localhost kernel: [<ffffffff8116d21e>]
↪oom_kill_process+0x24e/0x3b0
Oct 14 10:22:40 localhost kernel: [<ffffffff81088e4e>] ?
↪has_capability_noaudit+0x1e/0x30
Oct 14 10:22:40 localhost kernel: [<ffffffff811d4285>]
↪mem_cgroup_oom_synchronize+0x575/0x5a0
Oct 14 10:22:40 localhost kernel: [<ffffffff811d3650>] ?
↪mem_cgroup_charge_common+0xc0/0xc0
Oct 14 10:22:40 localhost kernel: [<ffffffff8116da94>]
↪pagefault_out_of_memory+0x14/0x90
Oct 14 10:22:40 localhost kernel: [<ffffffff8162f815>]
↪mm_fault_error+0x68/0x12b
Oct 14 10:22:40 localhost kernel: [<ffffffff816422d2>]
↪__do_page_fault+0x3e2/0x450
Oct 14 10:22:40 localhost kernel: [<ffffffff81642363>]
↪do_page_fault+0x23/0x80
Oct 14 10:22:40 localhost kernel: [<ffffffff8163e648>]
↪page_fault+0x28/0x30
Oct 14 10:22:40 localhost kernel: Task in /foo killed as
↪a result of limit of /foo
Oct 14 10:22:40 localhost kernel: memory: usage 4kB, limit
↪4kB, failcnt 8
Oct 14 10:22:40 localhost kernel: memory+swap: usage 4kB,
↪limit 9007199254740991kB, failcnt 0
Oct 14 10:22:40 localhost kernel: kmem: usage 0kB, limit
↪9007199254740991kB, failcnt 0
Oct 14 10:22:40 localhost kernel: Memory cgroup stats for /foo:
↪cache:0KB rss:4KB rss_huge:0KB mapped_file:0KB swap:0KB
↪inactive_anon:0KB active_anon:0KB inactive_file:0KB
↪active_file:0KB unevictable:0KB
Oct 14 10:22:40 localhost kernel: [ pid ]   uid  tgid total_vm
↪rss nr_ptes swapents oom_score_adj name
Oct 14 10:22:40 localhost kernel: [ 2687]     0  2687    28281
↪347     12        0             0 sh
Oct 14 10:22:40 localhost kernel: [ 2702]     0  2702    28281
↪50    7        0             0 sh
Oct 14 10:22:40 localhost kernel: Memory cgroup out of memory:
↪Kill process 2687 (sh) score 0 or sacrifice child
Oct 14 10:22:40 localhost kernel: Killed process 2702 (sh)
↪total-vm:113124kB, anon-rss:200kB, file-rss:0kB
Oct 14 10:22:41 localhost kernel: sh invoked oom-killer:
↪gfp_mask=0xd0, order=0, oom_score_adj=0
[ ... ]
```


请注意，内核的Out-Of-Mempry Killer（也叫做oom-killer 内存不足杀手）在应用程序达到4KB限制时就会介入。它会杀死应用程序，应用程序将不再运行，你可以通过输入以下命令进行验证：

```
$ ps -o cgroup 2687
CGROUP
```

### 使用libcgroup

之前描述的许多早期步骤都可以通过`libcgroup`包中提供的管理工具进行简化。例如，使用`cgcreate`二进制文件的单个命令即可创建sysfs条目和文件。

输入以下命令即可在`内存`子系统下创建一个叫做`foo`的控制组：

```
$ sudo cgcreate -g memory:foo
```


*注意：libcgroup提供了一种管理控制组中任务的机制。*

使用与之前相同的方法，你就可以开始设置内存阈值：

```
$ echo 50000000 | sudo tee
↪/sys/fs/cgroup/memory/foo/memory.limit_in_bytes
```


验证新配置的设置：

```
$ sudo cat memory.limit_in_bytes
50003968
```

使用`cgexec`二进制文件在控制组`foo`中运行应用程序：

```
$ sudo cgexec -g memory:foo ~/test.sh
```

使用它的进程ID - PID来验证应用程序是否在控制组和子系统（`内存`）下运行：

```
$  ps -o cgroup 2945
CGROUP
6:memory:/foo,1:name=systemd:/user.slice/user-0.slice/
↪session-1.scope
```


如果您的应用程序不再运行，并且您想要清理并删除控制组，则可以使用二进制文件`cgdelete`来执行此操作。要从`内存`控制器下删除控制组`foo`，请输入：

```
$ sudo cgdelete memory:foo
```

### 持久组

您也可以通过一个简单的配置文件和服务的启动来完成上述所有操作。您可以在`/etc/cgconfig.conf`文件中定义所有控制组名称和属性。以下为`foo`组添加了一些属性：

```
$ cat /etc/cgconfig.conf
#
Copyright IBM Corporation. 2007#
Authors:     Balbir Singh <balbir@linux.vnet.ibm.com>This program is free software; you can redistribute itand/or modify it under the terms of version 2.1 of the GNULesser General Public License as published by the FreeSoftware Foundation.#
This program is distributed in the hope that it would beuseful, but WITHOUT ANY WARRANTY; without even the impliedwarranty of MERCHANTABILITY or FITNESS FOR A PARTICULARPURPOSE.#
#
By default, we expect systemd mounts everything on boot,so there is not much to do.See man cgconfig.conf for further details, how to creategroups on system boot using this file.group foo {
cpu {
cpu.shares = 100;
}
memory {
memory.limit_in_bytes = 5000000;
}
} 
```

`cpu.shares`选项定义了该组的CPU优先级。默认情况下，所有组都继承1024 shares（CPU share指的是控制组中的任务被分配到的CPU的 time的优先级，即值越大，分配到的CPU time越多，这个值需大于等于2），即100%的CPU time（CPU time是CPU用于处理一个程序所花费的时间）。通过将`cpu.shares`的值降低到更保守的值（如100），这个组将会被限制只能使用大概10%的CPU time。

就如之前讨论的，在控制组中运行的进程也可以被限制它能访问的CPUs（内核）的数量。将以下部分添加到同一个配置文件`cgconfig.conf`中组名底下。

```
cpuset {
cpuset.cpus="0-5";
} 
```

有了这个限制，这个控制组会将应用程序绑定到到0核到5核——也就是说，它只能访问系统上的前6个CPU核。

接下来，您需要使用`cgconfig`服务加载此配置。首先，启用cgconfig以在系统启动时能够加载上述配置：

```
$ sudo systemctl enable cgconfig
Create symlink from /etc/systemd/system/sysinit.target.wants/
↪cgconfig.service
to /usr/lib/systemd/system/cgconfig.service.
```

现在，启动`cgconfig`服务并手动加载相同的配置文件（或者您可以跳过此步骤直接重启系统）：

```
$ sudo systemctl start cgconfig
```


在控制组`foo`下启动该应用程序并将其绑定到您设置的内存和CPU限制：

```
$ sudo cgexec -g memory,cpu,cpuset:foo ~/test.sh &
```


除了将应用程序启动到预定义的控制组之外，其余所有内容都将在系统重新启动后持续存在。但是，您可以通过定义依赖于`cgconfig`服务的启动初始脚本来启动该应用程序，自动执行该过程。

### 总结

通常来说，限制一个机器上一个或者多个任务的权限是必要的。控制组提供了这项功能，通过使用它，您可以对一些特别重要或无法控制的应用程序实施严格的硬件和软件限制。如果一个应用程序没有设置上限阈值或限制它可以在系统上消耗的内存量，cgroups可以解决这个问题。如果另一个应用程序没有CPU上的限制，那么cgroups可以再一次解决您的问题。您可以通过cgroup完成这么多工作，只需花一点时间，您就可以使用你的操作系统环境恢复稳定性，安全性和健全性。

在这个系列的第二篇中，我将会把焦点从控制组转移到Linux容器等技术是如何使用控制组的。

**原文链接：[Everything You Need to Know about Linux Containers, Part I: Linux Control Groups and Process Isolation](https://www.linuxjournal.com/content/everything-you-need-know-about-linux-containers-part-i-linux-control-groups-and-process)**

