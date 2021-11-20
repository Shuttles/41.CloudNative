# Kubeadm介绍

1. 直到 2017 年，在志愿者的推动下，社区才终于发起了一个独立的部署工具，名叫：[kubeadm](https://github.com/kubernetes/kubeadm)。

2. 这个项目的目的，就是要让用户能够通过这样**两条指令**完成一个 Kubernetes 集群的部署：

   ```shell
   # 创建一个 Master 节点
   $ kubeadm init
    
   # 将一个 Node 节点加入到当前集群中
   $ kubeadm join <Master 节点的 IP 和端口 >
   ```

3. 事实上，在 Kubernetes 早期的部署脚本里，确实有一个脚本就是==**用 Docker 部署 Kubernetes 项目的**==，这个脚本相比于 SaltStack 等的部署方式，也的确简单了不少。

   但是，**这样做会带来一个很麻烦的问题，即：如何容器化 kubelet。**

   `kubelet` 是 Kubernetes 项目**用来操作 Docker 等容器运行时的核心组件**。可是，除了跟容器运行时打交道外，`kubelet `在**配置容器网络、管理容器数据卷时**，都需要**直接操作宿主机**。

   而如果现在 `kubelet` **本身就运行在一个容器里**，**那么直接操作宿主机就会变得很麻烦。**对于网络配置来说还好，kubelet 容器可以通过不开启 Network Namespace（即 Docker 的 host network 模式）的方式，直接共享宿主机的网络栈。**可是，要让 kubelet 隔着容器的 Mount Namespace 和文件系统，操作宿主机的文件系统，就有点儿困难了**。

4. 正因为如此，kubeadm 选择了一种妥协方案：

   > ==把 kubelet 直接运行在宿主机上，然后使用容器部署其他的 Kubernetes 组件。==

5. 所以，你使用 kubeadm 的第一步，是在机器上手动安装 kubeadm、kubelet 和 kubectl 这三个二进制文件。







# Kubeadm init(master)

1. 当你执行 kubeadm init 指令后，**kubeadm 首先要做的，是一系列的检查工作，以确定这台机器可以用来部署 Kubernetes**。这一步检查，我们称为“Preflight Checks”，它可以为你省掉很多后续的麻烦。

2. **在通过了 Preflight Checks 之后，kubeadm 要为你做的，是生成 Kubernetes 对外提供服务所需的各种证书和对应的目录。**

   Kubernetes 对外提供服务时，除非专门开启“不安全模式”，否则都要通过 HTTPS 才能访问 kube-apiserver。这就需要为 Kubernetes 集群配置好证书文件。

   kubeadm 为 Kubernetes 项目生成的证书文件都放在 Master 节点的 /etc/kubernetes/pki 目录下。在这个目录下，最主要的证书文件是 ca.crt 和对应的私钥 ca.key。

3. **证书生成后，kubeadm 接下来会为其他组件生成访问 kube-apiserver 所需的配置文件**。这些文件的路径是：/etc/kubernetes/xxx.conf：

   这些文件里面记录的是，当前这个 Master 节点的服务器地址、监听端口、证书目录等信息。这样，对应的客户端（比如 scheduler，kubelet 等），可以直接加载相应的文件，使用里面的信息与 kube-apiserver 建立安全连接。

4. **接下来，kubeadm 会<u>为 Master 组件生成 Pod 配置文件</u>**。Kubernetes 有三个 Master 组件 `kube-apiserver`、`kube-controller-manager`、`kube-scheduler`，而**它们都会被使用 `Pod` 的方式部署起来。**

   在 kubeadm 中，Master 组件的 YAML 文件会被生成在 /etc/kubernetes/manifests 路径下。比如，kube-apiserver.yaml

   而一旦这些 YAML 文件出现在**被 `kubelet` 监视**的 /etc/kubernetes/manifests 目录下，==**`kubelet`** 就会自动创建这些 YAML 文件中定义的 `Pod`，即 Master 组件的容器。==

   Master 容器启动后，kubeadm 会通过检查 localhost:6443/healthz 这个 Master 组件的**健康检查 URL**，等待 Master 组件完全运行起来。

5. **然后，kubeadm 就会为集群生成一个 bootstrap token**。在后面，只要持有这个 token，任何一个安装了 kubelet 和 kubadm 的节点，都可以通过 kubeadm join 加入到这个集群当中。

   这个 token 的值和使用方法会，会在 kubeadm init 结束后被打印出来。

6. **在 token 生成之后，kubeadm 会将 ca.crt 等 Master 节点的重要信息，通过 ConfigMap 的方式保存在 Etcd 当中，供后续部署 Node 节点使用**。这个 ConfigMap 的名字是 cluster-info。

7. **kubeadm init 的最后一步，就是安装默认插件**。Kubernetes 默认 kube-proxy 和 DNS 这两个插件是必须安装的。它们分别用来提供整个集群的服务发现和 DNS 功能。其实，这两个插件也只是两个容器镜像而已，所以 kubeadm 只要用 Kubernetes 客户端创建两个 Pod 就可以了。



# kubeadm join(worker)

1. 这个流程其实非常简单，`kubeadm init` 生成 `bootstrap token `之后，你就可以在任意一台安装了 kubelet 和 kubeadm 的机器上执行 `kubeadm join` 了。

   可是，为什么执行 kubeadm join 需要这样一个 `token` 呢？

2. 因为，**任何一台机器想要成为 Kubernetes 集群中的一个节点，就必须在集群的 kube-apiserver 上注册**。可是，要想跟 apiserver 打交道，这台机器就必须要获取到相应的证书文件（CA 文件）。可是，为了能够一键安装，我们就不能让用户去 Master 节点上手动拷贝这些文件。

3. 所以，kubeadm 至少需要发起一次“**不安全模式**”的访问到 kube-apiserver，从而拿到保存在 ConfigMap 中的 cluster-info（它保存了 APIServer 的授权信息）。==而 bootstrap token，扮演的就是这个过程中的**安全验证**的角色。==

4. 只要有了 cluster-info 里的 kube-apiserver 的地址、端口、证书，kubelet 就可以以“安全模式”连接到 apiserver 上，这样一个新的节点就部署完成了。



# 配置kubeadm的部署参数

1. 我在前面讲解了 kubeadm 部署 Kubernetes 集群最关键的两个步骤，kubeadm init 和 kubeadm join。相信你一定会有这样的疑问：kubeadm 确实简单易用，可是我又该如何定制我的集群组件参数呢？

   比如，我要指定 kube-apiserver 的启动参数，该怎么办？

   在这里，我强烈推荐你在使用 kubeadm init 部署 Master 节点时，使用下面这条指令：

   ```shell
   kubeadm init --config kubeadm.yaml
   ```

   这时，**你就可以给 kubeadm 提供一个 YAML 文件**（比如，kubeadm.yaml）

2. 通过制定这样一个部署参数配置文件，**你就可以很方便地在这个文件里填写各种自定义的部署参数了。**

3. 而这个 YAML 文件提供的可配置项远不止这些。比如，你还可以修改 `kubelet` 和 `kube-proxy` 的配置，修改 Kubernetes 使用的基础镜像的 URL（默认的`k8s.gcr.io/xxx`镜像 URL 在国内访问是有困难的），指定自己的证书文件，指定特殊的容器运行时等等。这些配置项，就留给你在后续实践中探索了。



# 总结

1. 你可以看到，kubeadm 的设计非常简洁。并且，它在实现每一步部署功能时，都在最大程度地重用 Kubernetes 已有的功能，这也就使得我们在使用 kubeadm 部署 Kubernetes 项目时，非常有“**原生**”的感觉，一点都不会感到突兀。

2. kubeadm 能够用于生产环境吗？

   到目前为止（2018 年 9 月），这个问题的答案是：不能。

3. 因为 kubeadm 目前最欠缺的是，**一键部署一个高可用的 Kubernetes 集群，即：Etcd、Master 组件都应该是多节点集群，而不是现在这样的单点**。这，当然也正是 kubeadm 接下来发展的主要方向。

   另一方面，Lucas 也正在积极地把 kubeadm phases 开放给用户，即：**用户可以更加自由地定制 kubeadm 的每一个部署步骤。这些举措，都可以让这个项目更加完善，我对它的发展走向也充满了信心。**

4. 当然，如果你有部署规模化生产环境的需求，我推荐使用[kops](https://github.com/kubernetes/kops)或者 SaltStack 这样更复杂的部署工具。

5. kubeadm优点

   - 一方面，作为 Kubernetes 项目的原生部署工具，**kubeadm 对 Kubernetes 项目特性的使用和集成，确实要比其他项目“技高一筹”，非常值得我们学习和借鉴**；
   - 另一方面，kubeadm 的部署方法，**不会涉及到太多的运维工作，也不需要我们额外学习复杂的部署工具。而它部署的 Kubernetes 集群，跟一个完全使用二进制文件搭建起来的集群几乎没有任何区别**。

   因此，使用 kubeadm 去部署一个 Kubernetes 集群，**对于你理解 Kubernetes 组件的工作方式和架构，最好不过了**。