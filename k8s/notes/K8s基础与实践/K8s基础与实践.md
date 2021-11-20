# 1.背景

1. Kubernetes 的优势就在于：从设计之初就**以统一的方式定义任务之间的各种关系**， 并且为将来支持更多种类的关系留有余地，能够按照用户的意愿和整个系统的规则，完全自动化地处理好容器之间的各种关系，这个过程也就是**编排**。而其他很多集群管理项目（比如 Yarn、Mesos，以及 Swarm）擅长的，把一个容器按规则，放置在某个最佳节点上运行的功能则称为**调度**。
2. 容器本质是**进程**，那 Kubernetes 作为具有普遍意义的容器编排工具，就是**云操作系统**，TCOS 是星环为大数据分析平台量身定做的容器云操作系统，除了保持与开源社区的同步外，对容器平台的能力进行了更丰富的扩展，可以为上层大数据业务容器化运行以及应用云端化提供关键保障。



# 2.基础概念

## 2.1K8s概述

### 1.设计理念

1. 理解 Kubernetes 设计理念是学习 Kubernetes 的前提。我们需要聚焦在两个问题：

   - **如何处理应用与应用之间的关系？**
   - **如何恰当的容器化一个应用？**

2. 第一个问题，应用与应用之间的关系可以细化为容器间的关系，具体来说是两类：**一类关系是 “紧密交互” 的**，即：这些应用之间需要频繁交互、访问，或者会直接通过本地文件进行信息交换。Kubernetes 把这类关系涉及到的一组容器划分为一个 **“ <u>Pod</u> ”**（ Kuberntes 最小调度单位，后文详述），在这里面可以进行高效信息交换。

   **另一类关系则是常见的应用间的普通访问**，Kuberntes 通过定义 “<u>**服务对象**</u>” 来描述。比如 web 应用和数据库应用的交互，涉及到固定 IP 地址和端口以负载均衡的方式访问，就产生了 Service 对象来处理；加密授权的关系需求，则可由 Secret 对象解决等等。

3. 第二个问题，以 Pod 为基础为解决不同的场景需求衍生出了不同的解决方案，也就是**基于 Pod 改进后的对象资源，称为 “编排对象”**。比如被称为 DaemonSet 的对象资源，它可以像守护进程一样在每个宿主机上有且只能有一个pod副本；再比如 CronJob ，它专门用来描述定时任务等等。

4. 如下图所示，由 “Pod” 产生各类 “**编排对象**”，再为解决各种关系问题产生了类似Service、Ingress 等 “**服务对象** ”。

   ![img](https://qqadapt.qpic.cn/txdocpic/0/7378f19e739f0f6d9494ac8facd918fc/0?w=1920&h=1080)

5. 至此，总结下 Kubernetes 的通用使用方法：

   - 通过 “**编排对象**”，比如上文提到的 Pod、DaemonSet、Job 等来**描述被管理的应用**；
   - 定义一些 “**服务对象**”，比如 Service、Secret 等来**模拟平台对 “编排对象” 的管理**。

   以上提到的两种对象被称为 **API 对象（ API Object ）**，这种使用方法被称为 **“声明式 API ”**：**我们只需要描述我们期望达到的状态，而无需去在意具体的实现过程**。Kubernetes 里通常是用 YAML 文件来实现的，可以联想下最常见的声明式例子：SQL 来帮助理解 。



### 2.K8s架构

![img](https://qqadapt.qpic.cn/txdocpic/0/9b673f772555827739c0ec3a90182dd8/0?w=1500&h=685)

1. Kubernetes 由 Master 控制节点和 Node 计算节点两种节点组成。
2. Master 节点是由负责 API 服务的 **kube-apiserver**（操作资源的唯一入口，集群控制的入口进程）、负责调度的 **kube-scheduler**（负责pod调度），以及负责容器编排的 **kube-controller-manager**的组成（资源对象的自动化控制中心）。整个集群的持久化数据，则由 kube-apiserver 处理后保存在 **Etcd** 中。
3. Node 节点 运行 **kubelet**（负责pod创建启停，实现集群管理基本功能）, **kube-proxy**（实现Service的通信和负载均衡机制）。
4. 各组件的相互协作流程可查看 **2.4 核心组件运行机制。**



## 2.Pod

1. **Pod是什么？**

   Pod，其实是**一组共享了某些资源的容器**。它只是一个<u>逻辑概念</u>。具体的说：Pod 里的所有容器，共享的是同一个 Network Namespace，并且可以声明共享同一个 Volume。

2. **为什么需要Pod？**

   我们在Docker基础里已经知道**容器的本质是一个进程。**

   在一个真正的操作系统里，进程并不是“孤苦伶仃”地独自运行的，而是以**进程组**的方式，“<u>有原则地”组织在一起。它们紧密协作共同完成职责</u>。

   Kubernetes 项目所做的，其实就是**将“进程组”的概念映射到了容器技术中**。<u>一组容器间也会有每个容器负责应用的一部分功能，紧密协作共同完成职责的情况，比如它们之间使用 localhost 或者 Socket 文件进行本地通信，发生频繁文件交换等</u>。

   **在调度时，这样的一组容器必须被部署在同一台机器上，应该被作为一个整体来调度**。所以，Kubernetes 将这样一组容器定义为一个 Pod，并把 Pod 作为**最小的调度单位**。

3. **Pod模型**

   在 docker 中，容器共享**网络和 Volume** 是非常简单的事情。对于容器A 和 容器B，使用一条命令就可以实现：

   ```
   $ docker run --net=B --volumes-from=B --name=A image-A ...
   ```

   但是，这样做的话，容器 B 就必须比容器 A 先启动，一个 Pod 里的多个容器就不是对等关系，而是拓扑关系了。

   所以，在 Kubernetes 项目里，Pod 的实现需要使用一个中间容器，这个容器叫作 Infra 容器。**在这个 Pod 中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起**。这样的组织关系，可以用下面这样一个示意图来表达：

   ![img](https://qqadapt.qpic.cn/txdocpic/0/42abeb3e4a03dc9eb2bd21677d50c6a2/0?w=490&h=665)

   这样的设计使得**Pod的生命周期只跟 infra 容器有关**，**而与用户容器 A 和 B 无关**。这也是为什么在 k8s 内，允许用户去单独更新 Pod 里的某一个镜像。即：**做这个操作，整个 Pod 既不会重建，也不会重启。**

   <u>同时，涉及到调度、网络、存储，以及安全相关的属性，我们只需要关注 Pod 级别</u>。比如，在开发网络插件时，只需要关注如何配置 Pod 的 Network Namespace（即 Infra 容器的 Network Namespace）， 而不是考虑每一个用户容器如何使用网络配置。

   

## 3.容器编排控制器模式



# 3.常用资源对象

在本章中，1-6 节讲述的是**编排对象**，它们使用不同策略编排 Pod。而 1-6 节都在为 第7节的 **服务对象** Service 作铺垫。

8-9 节讲述的是 Pod 的存储和配置对象。

第 10 节讲述的是权限控制对象。

## 3.1编排对象

### 1.ReplicaSet

1. ReplicaSet 是 Kubernetes 系统中的核心核心概念之一。**作为一个编排对象，它通过“控制器模式”，保证集群中它所控制的 Pod 的个数永远等于用户期望的个数**。

2. 在 ReplicaSet 这个 YAML 文件中，最重要的字段就是 **spec.replicas,** **spec.selector**。

   - `spec.replicas` - 指定了期望的 Pod 个数
   - `spec.selector` - 用于筛选目标 Pod

   由此 YAML 创建的 ReplicaSet 将根据 spec.selector 来统计具有 app: nginx 标签的 Pod 的数量，当数量不足 3 个时，将根据 template 模板创建Pod。所以**请确保spec.selector 和 template.metadata.labels 中的标签一致**，否则 ReplicaSet 将不停创建 Pod。





### 2.Deployment

1. Deployment 也是一个编排对象。它可以被认为是 **ReplicaSet 的升级**，**同样保证了集群中的Pod数量，但他实现了“滚动更新”这个非常重要的功能**。

2. Deployment 控制器==实际操纵的是 **ReplicaSet** 对象，而不是 Pod 对象==。

3. Deployment 实际上是一个两层控制器。首先，它**通过 ReplicaSet 的个数来描述应用的版本**（一个 ReplicaSet 对应 应用的一个版本）；然后，它再**通过 ReplicaSet 的属性**（比如 replicas 的值），**来保证 Pod 的副本数量**。

   ![img](https://qqadapt.qpic.cn/txdocpic/0/a7198a5546ddb681be69632b49227c60/0?w=341&h=321)

   



### 3.StatefulSet

1. Deployment 假设的是一个应用的所有 Pod， 是完全一样的，是**无状态的**。但是有些应用的多个 Pod 之间往往有顺序，主从关系，或者 Pod 对外部数据有依赖关系，这些应用是**有状态的**。所以，Kubernetes 扩展出了 `StaefulSet` 这个编排对象。

2. Kubernetes 把有状态应用分为了两种情况：

   1. **拓扑状态**：Pod **必须按照某些顺序**启动，更新或重新创建。且每个节点都有唯一的网络标识，即使 Pod 重新创建后的网络标识也应该和原来一样。
   2. **存储状态**：Pod 所存储的数据应该是持久的。即使 Pod 重新创建后，读取到的数据也应该是同一份。

3. **StatefulSet 解决方式**：

   1. StatefulSet 会对Pod进行编号，并且按照编号顺序逐一完成创建、更新等工作。StatefulSet 通过 Headless Service 的方式，为每个 Pod 创建了一个固定并且稳定的 DNS 记录，来作为它的访问入口。
   2. StatefulSet 通过 PV (Persistent Volume) 和 PVC (Persistent Volume Claim) 来实现持久化存储。

   使用场景：MySQL, MongoDB, Akka, ZooKeeper 等

### 4.DaemonSet

1. 顾名思义，DaemonSet 的主要作用，<u>是让你在 Kubernetes 集群里，运行一个 Daemon Pod。</u> 所以，这个 Pod 有如下四个特征：
   1. 这个 Pod **运行在** Kubernetes 集群里的**每一个节点（Node）上**；
   2. **每个节点上只有一个**这样的 Pod 实例；
   3. 当有新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉；当节点上的 Pod 出现异常时，会由 Daemonset 进行恢复。
   4. 删除 DaemonSet 将会删除它创建的所有 Pod
2. `DaemonSet `**使用场景**：
   1. 各种**网络插件**的 Agent 组件，都必须运行在每一个节点上，用来处理这个节点上的容器网络；
   2. 各种**存储插件**的 Agent 组件，也必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的 Volume 目录；
   3. 各种**监控组件和日志组件**，也必须运行在每一个节点上，负责这个节点上的监控信息和日志搜集。



==Deployment，StatefulSet，DamonSet 编排的对象都是长作业，应用一旦运行起来，它的容器进程会一直保持 Running 状态。== 而Job和CronJob编排的对象不是长作业

### 5.Job

1. Job 负责==**计算业务**==：
   1. Job 所控制的 Pod 副本都是**短暂运行**的（成功完成计算任务 Pod就停止，不会再重启）**。**
   2. Job 所控制的 Pod 副本的工作模式能够**多实例并行计算。**
   3. Job 可以根据依赖关系，保证任务的顺序进行。
2. Job 会创建**一个或者多个 Pods**，并确保指定数量的 Pods 成功终止。 随着 Pods 成功结束，Job 跟踪记录成功完成的 Pods 个数。 **当数量达到指定的成功个数阈值时，任务（即 Job）结束**。 删除 Job 的操作会清除所创建的全部 Pods。
3. 一种简单的使用场景下，可以**创建一个 Job 对象以便以一种可靠的方式运行某 Pod 直到完成**。 当第一个 Pod 失败或者被删除（比如因为节点硬件失效或者重启）时，Job 对象会启动一个新的 Pod。



### 6.CronJob

1. CronJob是**基于时间调度的Job**，它对于**创建周期性的、反复重复的任务很有用**，例如执行<u>数据备份或者发送邮件</u>。
2. CronJob 与 Job 的关系，正如同 Deployment 与 ReplicaSet 的关系一样：**CronJob 仅负责创建与其调度时间相匹配的 Job，而 Job 又负责管理其代表的 Pod**。
3. 值得注意的是：**Job应该是幂等的**。



## 3.2服务对象



### 1.Service

1. Service 是 Kubernetes 中的核心概念之一。我们之前讲述的**编排对象** `ReplicaSet`, `Deployment` 等都是在为 `Service` **作铺垫。**

2. 我们知道 **Pod 需要对外提供服务**，但是 P**od 的 IP 地址等信息不是固定的**，通过 Pod IP+port 来访问服务是很困难的。所以，K8S 提出了 Service 服务对象。

3. Service 的主要作用就是，==**作为一个服务的访问入口，用户可以通过这个入口地址访问其背后的一组由Pod副本组成的集群实例**==。<u>用户通过访问固定的 IP+port，Service 就会根据负载均衡机制，将用户请求发送到它所管理的某一个 Pod 中</u>。

4. 我们用一副图来表示 Service, ReplicaSet 和 Pod 之间的关系。

   ​                 ![img](https://qqadapt.qpic.cn/txdocpic/0/1b635f90cbef7c11d0e505dfc7a1a6fc/0?w=1413&h=619)

   可以看见，前端应用（Pod）**通过访问 Service 来获取后端应用提供的服务**。`Service` 和 `ReplicaSet` 都通过 `Label `来选择它们所管理的 Pod。 
   
   ==RS 等**编排对象**的作用实际上是**保证 Service 的服务能力和服务质量始终符合预期标准**==。        
   
   



## 3.3存储对象volume

1. 一个 Pod 中的容器可以声明共享同一个 Volume。Volume实际上就是一个 Pod 中能够被多个容器访问的**共享目录**。

2. 在 K8S 中，Volume 是定义在 **Pod 层级**的，**Pod 中的容器只需要声明挂载这个 Volume 就可以开始共享这个目录。**

   由于 Volume 是定义在 Pod 层级的，所以 **Volume 的生命周期和 Pod 的生命周期相同**，即使 Pod 中的容器重启或停止了，Volume 中的数据也不会丢失，重启后的容器仍然能从该 Volume 中读取到之前的数据。

3. K8S 支持多种类型的 Volume。

   1. **emptyDir**：临时 Volume，当 Pod 被移除后，该 Volume 中的数据将被永久删除
   2. **hostPath**：使用宿主机上的目录作为 Volume，即使 Pod 被移除了，其中数据也会被保存在宿主机上，但是它们不能被”迁移“到其他节点上。
   3. **远程存储服务器**：NFS，rbd，gcePersistentDisk等。使用远程存储服务器作为 Volume，即使 Pod 被移除了，数据会被永久保存在远程服务器上。



## 3.4配置对象

### 1.ConfigMap 和 Secret 

1. ConfigMap 和 Secret 是两种特殊的 Volume，它们属于 K8S 中的**服务对象**。它们不是为了存储数据，也不是为了容器间的数据交换，而是为了**给容器提供预先定义好的数据**。
2. ConfigMap 和 Secret 对象会预先存放进 etcd 中，当容器声明挂载这些 Volume 后，就可以从相应的 mountPath 中读取到这些信息了。
3. **ConfigMap** 中存放的是不需要加密的，应用所需的**配置信息**，比如挂载 Pod 用的配置文件、环境变量、命令行参数等。
4. **Secret** 中存放的是加密的数据，比如数据库的用户名和密码，token等。其中的敏感数据采用 base-64 编码保存，相比存储在明文中的 ConfigMap 更加规范与安全。



### 2.ServiceAccount

1. **ServiceAccount** 对象的作用，就是 K8S 系统内置的一种”服务账户“，它是 K8S 进行权限分配的对象，解决了 **Pod 在集群中的身份认证问题。**
2. 它其实是一种**特殊的 Secret**，==**应用 Pod 必须使用 ServiceAccount 中保存的授权信息，才能合法地访问 API Server**==。如果 Pod 没有声明 ServiceAccountName，Kubernetes 会自动在它的 Namespace 下创建一个名叫 default 的默认 ServiceAccount，然后分配给这个 Pod。





## 3.5权限控制对象RBAC

1. Kubernetes 中所有的 API 对象，都保存在 Etcd 里。可是，对这些 API 对象的操作，却一定都是通过访问 kube-apiserver 实现的。其中一个非常重要的原因，就是你**需要 APIServer 来帮助你做授权工作**。
2. 而在 Kubernetes 项目中，**负责完成授权（Authorization）工作的机制，就是 RBAC**：基于角色的访问控制（Role-Based Access Control）。
3. 在 RBAC 中，有三个最基本的概念。
   1. **Role**：角色，它其实是一组**规则**，定义了一组对 Kubernetes API 对象的操作权限。
   2. **Subject**：被作用者，既可以是“人”，也可以是“机器”，也可以是你在 Kubernetes 里定义的“用户”。
   3. **RoleBinding**：定义了“被作用者”和“角色”的绑定关系。
4. Role 本身就是一个 API 对象
5. RoleBinding 也是一个 API 对象。它指定了规则的被作用者。
6. **对于非 Namespaced（Non-namespaced）对象（比如：Node），或者，某一个 Role 想要作用于所有的 Namespace 的时候，**我们可以使用 **ClusterRole** 和 **ClusterRoleBinding** 进行授权。。这两个 API 对象的用法跟 Role 和 RoleBinding 完全一样。只不过，它们的定义里，没有 Namespace 字段。





# 参考资料

1. K8s基础与实践

   https://docs.qq.com/doc/DVEticm93aWhJcWRP