



# 1.背景

首先一起来看一下需求来源。大家应该都有过这样的经验，就是用一个容器镜像来启动一个 container。要启动这个容器，其实有很多需要配套的问题待解决：

1. 第一，比如说**一些可变的配置**。因为我们不可能把一些可变的配置写到镜像里面，当这个配置需要变化的时候，可能需要我们重新编译一次镜像，这个肯定是不能接受的；
2. 第二就是一些**敏感信息的存储和使用**。比如说应用需要使用一些密码，或者用一些 token；
3. 第三就是我们**容器要访问集群自身**。比如我要访问 kube-apiserver，那么本身就有一个身份认证的问题；
4. 第四就是容器在节点上运行之后，它的**资源需求**；
5. 第五个就是容器在节点上，它们是共享内核的，那么它的一个**安全管控**怎么办？
6. 最后一点我们说一下**容器启动之前的一个前置条件检验**。比如说，一个容器启动之前，我可能要确认一下 DNS 服务是不是好用？又或者确认一下网络是不是联通的？那么这些其实就是一些前置的校验。



## Pod 的配置管理

在 Kubernetes 里面，它是怎么做这些配置管理的呢？如下图所示：

![img](https://edu.aliyun.com/files/course/2021/04-02/170223f594d6059219.png)

1. 可变配置就用 **ConfigMap**；
2. 敏感信息是用 **Secret**；
3. 身份认证是用 **ServiceAccount** 这几个独立的资源来实现的；
4. 资源配置是用 **Resources**；
5. 安全管控是用 **SecurityContext**；
6. 前置校验是用 **InitContainers** 这几个在 spec 里面加的字段，来实现的这些配置管理。





# 2.ConfigMap

## 1.介绍

1. 它其实主要是==**管理一些可变配置信息**==，比如说我们应用的一些**配置文件**，或者说它里面的一些**环境变量**，或者一些**命令行参数**。
2. 它的好处在于它可以**<u>让一些可变配置和容器镜像进行解耦</u>**，这样也保证了容器的**可移植性**。看一下下图中右边的编排文件截图。
3. ![img](https://edu.aliyun.com/files/course/2021/04-02/170250a38e19965856.png)
4. 这是 ConfigMap 本身的一个定义，它包括两个部分
   + 一个是 ConfigMap 元信息，我们关注 name 和 namespace 这两个信息。
   + 接下来这个 data 里面，可以看到它管理了两个配置文件。它的结构其实是这样的：从名字看ConfigMap中包含Map单词，Map 其实就是 key:value，key 是一个文件名，value 是这个文件的内容。



## 2.创建

1. 看过介绍之后，再具体看一下它是怎么创建的。

   我们推荐用 **kubectl** 这个命令来创建，它带的参数主要有两个：一个是指定 **name**，第二个是 **DATA**。其中 DATA 可以通过指定文件或者指定目录，以及直接指定键值对，下面可以看一下这个例子。

2. 指定文件的话，**文件名就是 Map 中的 key**，**文件内容就是 Map 中的 value**。然后指定键值对就是指定数据键值对，即：key:value 形式，直接映射到 Map 的key:value。



## 3.使用

![img](https://edu.aliyun.com/files/course/2021/04-02/170411bacf54671201.png)

如上图所示，主要是在 pod 里来使用 ConfigMap：

1. 第一种是**环境变量**。环境变量的话通过 valueFrom，然后 ConfigMapKeyRef 这个字段，下面的 name 是指定 ConfigMap 名，key 是 ConfigMap.data 里面的 key。这样的话，在 busybox 容器启动后容器中执行 env 将看到一个 SPECIAL_LEVEL_KEY 环境变量；

2. 第二个是**命令行参数**。命令行参数其实是第一行的环境变量直接拿到 cmd 这个字段里面来用；

3. 最后一个是**通过 volume 挂载的方式直接挂到容器的某一个目录下面去**。上面的例子是把 special-config 这个 ConfigMap 里面的内容挂到容器里面的 /etc/config 目录下，这个也是使用的一种方式。

   （==不是很懂，声明volume不是给container放数据的吗？==）



## 4.注意点

1. ConfigMap **文件的大小**。虽然说 ConfigMap 文件没有大小限制，但是在 ETCD 里面，数据的写入是有大小限制的，现在是限制在 1MB 以内；
2. 第二个注意点是 pod 引入 ConfigMap 的时候，必须是**相同的 Namespace 中的 ConfigMap**，前面其实可以看到，ConfigMap.metadata 里面是有 namespace 字段的；
3. 第三个是 pod 引用的 ConfigMap。假如这个 ConfigMap 不存在，那么这个 pod 是无法创建成功的，其实这也表示在创建 pod **前**，**必须先把要引用的 ConfigMap 创建好**；
4. 第四点就是**使用 envFrom 的方式**。把 ConfigMap 里面所有的信息导入成环境变量时，如果 ConfigMap 里有些 key 是无效的，比如 key 的名字里面带有数字，那么这个环境变量其实是不会注入容器的，它会被忽略。**但是这个 pod 本身是可以创建的**。这个和第三点是不一样的方式，是 ConfigMap 文件存在基础上，整体导入成环境变量的一种形式；
5. 最后一点是：什么样的 pod 才能使用 ConfigMap？这里只有通过 K8s api 创建的 pod 才能使用 ConfigMap，比如说通过用命令行 kubectl 来创建的 pod，肯定是可以使用 ConfigMap 的，但其他方式创建的 pod，比如说 kubelet 通过 manifest 创建的 **static pod，它是不能使用 ConfigMap 的**。