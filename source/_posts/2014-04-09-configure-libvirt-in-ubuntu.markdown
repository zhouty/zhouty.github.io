---
layout: post
title: "Ubuntu12.04编译配置libvirt小记"
date: 2014-04-09 21:51:25 +0800
comments: true
categories: virtualization
---

轰轰烈烈的项目终于告一段落了，不得不说好久不写代码现在写几行最简单的功能就bug无数。从这个角度来看的话做做项目也是不错的，至少不会让手过于生疏。我们主要是基于[libvirt][1]用python开发了一款小型的分布式虚拟机管理系统，所以在这里先简单介绍下libvirt。

libvirt是一个十分强大的虚拟机管理工具集，可以用于开发上层的管理工具比如linux上的virt-manager等。它主要有三部分组件组成，分别是与hypervisor交互的守护进程libvirtd， 为上层应用提供接口的libvirt API，以及一款基于libvirt的命令行工具virsh。libvirt支持多种hypervisor比如KVM/QEMU, Xen, LXC, VirtualBox，VMWare, Parallels, Hyper-V等等，并且现在已经有多种[语言支持][2]，比如常用的C#, JAVA, Python, PHP, Ruby等，十分的方便。

不过配置libvirt的过程还是有些蛋疼的。libvirt现在由Redhat公司维护，所以网上现有的配置方法大部分都是基于CentOS的，好多问题只能自己摸索。还好有给力的[聪聪大哥][3]，帮我解决了很多问题，现在就把安装过程和碰到的问题列举一下，以便查阅。

<!--more-->  
首先在下载[源码][4],我用的是1.1.3版本的。新的版本一般会多一些功能，而且函数的很多参数都成了可选的。首先可以在libvirt目录下运行`./configure`，libvirt会检查系统当前缺少那些依赖。一般需要安装的依赖有以下几个：

    $sudo apt-get install libyajl-dev libxml++2.6-2 libxml++2.6-dev libdevmapper-dev libpciaccess-dev python-dev libnl-dev libpciaccess-dev
然后运行`make && make install`安装。这是应该能够在python目录下的dist-packages文件夹中看到一个名为libvirt.py的文件，这就是我们要用的libvirt python API了。如果没有的话就再运行`sudo apt-get install python-libvirt`安装一遍吧。

如果你以为到这里就大功告成了，那就大错特错了。当然你有可能上辈子积德行善，然后运行`sudo libvirtd -d`
没报错，那么恭喜你，完成一半任务了。当然我肯定是没这么好的运气了，碰到的如下错误：

>  libvirtd: error while loading shared libraries: libvirt-lxc.so.0: cannot open shared object file: No such file or directory

这时可以运行```ldconfig```，应该就没问题了，表明我们已经启动了libvirtd进程。然后运行`
virsh -c qemu:///system list`出现以下错误

> 
error: Failed to connect socket to '/var/run/libvirt/libvirt-sock': Permission denied
error: failed to connect to the hypervisor

后面的错误基本上是由于权限的问题了。首先修改libvirt的配置文件`/etc/libvirt/libvirtd.conf`中的几行，将group改为用户的group，权限改为770或777

> 
unix_sock_group = < group >  
unix_sock_ro_perms = < perms >  
unix_sock_rw_perms = < perms >

然后又碰到了另一个错误

> error: authentication failed: Authorization requires authentication but no agent is available.

libvirt采用polkit权限认证，需要添加文件 `/etc/polkit-1/localauthority/50-local.d/50-kvmcon-libvirt-access.pkla`，内容如下：

> [ libvirt management access ]
Identity=unix-group:libvirt
Action=org.libvirt.unix.manage
ResultAny=yes
ResultInactive=yes
ResultActive=yes

其中identity也可以用unix-user加用户名

> libvirt.libvirtError: authentication failed: polkit: Error checking for authorization org.libvirt.unix.manage: GDBus.Error:org.freedesktop.PolicyKit1.Error.Failed: Action org.libvirt.unix.manage is not registered

这时如果用"pkaction"这个命令列举所有polkit注册过的action的话是没有“org.libvirt.unix.manage"这个action的，所以应该在`/usr/share/polkit-1/actions/`目录下添加org.libvirt.unix.manage这个action，然后就应该可以了。
最后，如果在用libvirt启动虚拟机时报错

> error: Unknow OS type hvm

这时运`virsh capacities`是看不到后面的guest域的。我重新编译下qemu就可以了，如果再不行，就在libvirt的log里`/usr/local/var/log/libvirt/libvirtd.log或/var/log/libvirt/libvirtd.log`找找其他原因吧 : )







  [1]: http://libvirt.org/
  [2]: http://libvirt.org/bindings.html
  [3]: http://ytliu.info/
  [4]: http://libvirt.org/downloads.html
  
