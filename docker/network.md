# network mode
由上面的 Docker 原理可知 Docker 使用了 Linux 的 Namespaces 技术来进行资源隔离，如 PID Namespace 隔离进程，Mount Namespace 隔离文件系统，Network Namespace 隔离网络等。一个Network Namespace 提供了一份独立的网络环境(包括网卡、路由、Iptable规则)与其他的Network Namespace隔离，一个Docker容器一般会分配一个独立的Network Namespace。
当你安装Docker时，执行docker network ls会发现它会自动创建三个网络。

```
[root@server1 ~]$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
0147b8d16c64        bridge              bridge              local
2da931af3f0b        host                host                local
63d31338bcd9        none                null                local
```

我们在使用docker run创建Docker容器时，可以用 --net 选项指定容器的网络模式，Docker可以有以下4种网络模式：
网络模式使用注意host和宿主机共享网络none不配置网络bridgedocker默认，也可自创container容器网络连通，容器直接互联，用的很少
4.1 Host 模式
等价于Vmware中的桥接模式，当启动容器的时候用host模式，容器将不会虚拟出自己的网卡，配置自己的IP等，而是使用宿主机的IP和端口。但是容器的其他方面，如文件系统、进程列表等还是和宿主机隔离的。
4.2 Container 模式
Container 模式指定新创建的容器和已经存在的一个容器共享一个 Network Namespace，而不是和宿主机共享。新创建的容器不会创建自己的网卡，配置自己的IP，而是和一个指定的容器共享IP、端口范围等。同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。两个容器的进程可以通过lo网卡设备通信。
4.3 None 模式
None 模式将容器放置在它自己的网络栈中，并不进行任何配置。实际上，该模式关闭了容器的网络功能，该模式下容器并不需要网络（例如只需要写磁盘卷的批处理任务）。
4.4 Bridge 模式
Bridge 模式是 Docker 默认的网络设置，此模式会为每一个容器分配 Network Namespace、设置IP等。当Docker Server启动时，会在主机上创建一个名为docker0的虚拟网桥，此主机上启动的Docker容器会连接到这个虚拟网桥上。虚拟网桥的工作方式和物理交换机类似，主机上的所有容器就通过交换机连在了一个二层网络中。
Docker 会从RFC1918所定义的私有IP网段中，选择一个和宿主机不同的IP地址和子网分配给docker0，连接到 docker0 的容器从子网中选择个未占用 IP 使用。一般 Docker 会用 172.17.0.0/16 这个网段，并将172.17.0.1/16 分配给 docker0 网桥（在主机上使用ifconfig命令是可以看到docker0的，可以认为它是网桥的管理接口，在宿主机上作为一块虚拟网卡使用）。
网络配置的过程大致3步：

在主机上创建一对虚拟网卡 veth pair 设备。veth设备总是成对出现的，它们组成了一个数据的通道，数据从一个设备进入，就会从另一个设备出来。因此veth设备常用来连接两个网络设备。
Docker 将 veth pair 设备的一端放在新创建的容器中，并命名为eth0。另一端放在主机中，以veth65f9 这样类似的名字命名，并将这个网络设备加入到docker0网桥中，可以通过brctl show命令查看。
从 docker0 子网中分配一个IP给容器使用，并设置 docker0 的IP地址为容器的默认网关。

Bridge 模式下容器的通信

容器访问外部


假设主机网卡为eth0，IP地址10.10.101.105/24，网关10.10.101.254。从主机上一个IP为172.17.0.1/16 的容器中ping百度（180.76.3.151）。首先IP包从容器发往自己的默认网关 docker0，包到达docker0后，会查询主机的路由表，发现包应该从主机的 eth0 发往主机的网关10.10.105.254/24。接着包会转发给eth0，并从eth0发出去。这时Iptable规则就会起作用，将源地址换为 eth0 的地址。这样，在外界看来，这个包就是从10.10.101.105上发出来的，Docker容器对外是不可见的。

外部访问容器

创建容器并将容器的80端口映射到主机的80端口。当我们对主机 eth0 收到的目的端口为80的访问时候，Iptable规则会进行DNAT转换，将流量发往172.17.0.2:80，也就是我们上面创建的Docker容器。所以，外界只需访问10.10.101.105:80就可以访问到容器中的服务。