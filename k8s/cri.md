# CRI
K8s 已经在很早前的 1.20 版发布之前就宣布要移除内嵌在 kubelet 中的 docker shim 的代码，大概原因就是谷歌一直在 k8s 中使用一套叫做 CRI（container runtime interface）的规范，该规范旨在定义 k8s 如何更好地操作容器技术，该规范大概分为三部分：CRI Client，CRI Server，OCI Runtime。

简单来讲，就是在 kubelet 中放一个 grpc 的客户端，这个客户端要和一个 grpc 的服务端进行通信，这个 grpc 的服务端里头进行容器的，“拉起”，“销毁” 等动作的调用，而真正执行 “拉起”，“销毁” 等动作的代码由 OCI Runtime 实现。

或者再简单点，对应到实现来说：CRI Client 端有个实现就是 kubelet，CRI Server 端有个实现叫 Containerd，OCI Runtime 有好多实现其中有个叫 runc。然后把他们串起来就是：kubelet 在做完了一些准入校验，CSI 的存储卷挂载等操作之后，要去拉起一个 pod 了，在拉起 pod 的时候，就先启动一个 grpc 的客户端，然后与 Containerd 中的 grpc 的服务端通信，告诉它说要拉起一个 pod 了。然后 containerd 收到后会按照一定的流程去“拉镜像”，“创建 sandbox”，“创建 netns”，“启动容器”，“创建容器网络”，“把容器加入到 sandbox” 中等。containerd 基本上只负责调用（高级运行时），真正实现这些功能的地方在 OCI 的 runc（或其他低级运行时）中，有点像是通常服务端的 controller 和 service。

不同的低级运行时因为实现逻辑以及调用逻辑可能都不太一样，就比如 runc 是利用 namespace，kata 是直接起的 qemu 虚拟机，因此 containerd 到 OCI 之间还会有一层 shim，containerd 到 runc 有 runc-shim，到 kata 有 kata-shim。

所谓的废弃 docker，其实不是真的直接干掉 docker，而是上面我们说 CRI 规范要有个 grpc 的 client 和 server 端嘛，然后 server 端通知 oci 里头去创建 ns 或者创建网络等，但是对于 docker 来讲，docker 内置了创建网络，创建存储卷等功能，在 k8s 里，这些功能比如挂载存储功能由 CSI 实现，挂载网络由 CNI 实现等，所以 docker 很多功能 k8s 并不需要，因此 kubelet 内嵌了一个 docker-shim 作为 grpc 的 client 端用来屏蔽掉 docker 的一些内置功能后再和 containerd 通信，所以其实对于 k8s 来讲，这个 docker-shim 是个冗余的负担，因此所谓的“废弃”，指的是将 docker-shim 的代码从 kubelet 中移除，在减负的同时也能通过让第三方实现 CRI 规范中的东西而接入到 k8s 里。

## [CRI Client](./cri_client.md)
## [CRI Server](./cri_server.md)
## [OCI Runtime](./oci_runtime.md)