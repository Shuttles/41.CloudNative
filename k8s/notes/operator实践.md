



# 1.创建operator工程

```shell
mkdir $GOPATH/src/transwarp.io/tos/warpdrive-operator
cd $GOPATH/src/transwarp.io/tos/warpdrive-operator
# go mod init 
kubebuilder init --domain transwarp.io
# --domain: CRD域名，定义为transwarp.io
```

这一步创建了一个Go module工程，**引入必要的依赖**，并且**创建了一些模板文件**。





# 2.创建API

1. 所谓创建API，就是初始化Operator想要管理的**CRD**。以Warpdrive Operator为例，这个operator用于管理集群内部的存储资源，其中**WarpdrivePool CRD用于管理存储池资源**。

   ```shell
   kubebuilder create api --group tos --version v1alpha1 --kind WarpdrivePool
   # --group: CRD组名，定义为项目在gitlab所属group
   # --version: CRD版本，初始版本一般定义为v1alpha1，稳定release之后更改为v1
   # --kind: CRD名称
   ```

2. 创建API之后，项目目录下面增加了两个文件夹，分别是是CRD定义和controller模板文件：

   ```shell
   ├── apis
   │   └── tos
   │       └── v1alpha1
   │           ├── groupversion_info.go     // GV的通用元数据，用于CRD生成
   │           ├── warpdrivepool_types.go   // 自定义CRD的地方
   │           └── zz_generated.deepcopy.go  // rumtime.Object接口实现，主要是DeepCopy()
   ...
   ├── controllers
   │   └── tos
   │       ├── suite_test.go
   │       ├── warpdrivepool_controller.go   // controller核心逻辑实现
   ...
   
   ```

3. kubebuilder在生成的./api/v1alpha1/warpdrivepool_types.go为CRD创建了API模板，由以下部分组成：

   + **TypeMeta**: 描述API版本和类型
   + **ObjectMeta**: 描述CRD name, namespace, labels等信息
   + **Spec**： 定义CRD的期望状态（DSW, desired state of world）
   + **Status**：定义CRD的当前状态（ASW, actual state of world)

4. 在API定义的上方，除功能注释之外，还有// +kuberbuilder:格式的注释，这些注释称为**标记**（marker），代码生成工具**controller-gen**根据**标记**生成特定的功能代码。这里列出一些API定义常用的标记：

   + `kubebuilder:printcolumn`： 指定kubectl get < crd > 打印的字段
   + `kubebuilder:resource:scope=< string >,shortName=<[]string>`：指定CRD的简称和Scope，Scope可以定义为Namespaced和Cluster

   > 更多标记详见：https://book.kubebuilder.io/reference/markers.html

5. ==【总结】==在创建API时，开发者需要完成以下工作：

   + **完成CRD定义**，包括Spec和Status
   + 选择合适的CRD标记，**自定义API特性和接口**



# 3.初始化manager

> 💡总结：下面这些都是原理性的阐述，kubebuilder已经完整封装了上述功能，包括Client, Informer等，**<u>开发者一般不需要修改main.go</u>**



Manager可以理解为**controller的管理者**，封装了底层cache, client, informer的所有功能，包括

+ 负责**运行**所有的 Controllers；
+ 初始化**共享 caches**，包含 listAndWatch 功能；
+ 初始化 clients 用于**与 API Server 通信**。

其初始化和启动在main.go中定义。

main.go由init()和main()函数构成。init()将GV tosv1alpha1==注册到Scheme中==，而main()主要实现了以下功能：

1. **Manager初始化**
   + 创建**Cache**：为Scheme里的每个==GVK==创建相应的Informer，informersByGVK这个map维护GVK到Informer的映射关系，每个Informer会根据ListWatch函数对相应的GVK进行List和Watch
   + 创建**Client**：使用该Client的**读操作**从Cache读取；**写操作**使用k8s go-client直连api server
2. **Controller初始化**
   + 注册**Watch handler**到对应的GVK informer。这样一旦GVK里面的资源有变更都会触发Handler，将变更事件写入controller的事件队列（workQueue）中，最终触发Reconcile方法。
3. **Manager启动**
   + **Cache**启动：
     + Cache 的初始化核心是初始化所有的 Informer，Informer 的初始化核心是创建了 reflector 和内部 controller：reflector 负责监听 API Server 上指定的 GVK，将变更写入 delta 队列中，可以理解为变更事件的生产者；
     + 内部 controller 是变更事件的消费者，他会负责更新本地 indexer，以及计算出 CUD 事件推给我们之前注册的 Watch Handler。
   + **Controller**启动：启动 goroutine不断地查询队列（**workQueue**），如果有变更消息则触发到自定义的 Reconcile 逻辑。







# 4.初始化controller

> 💡总结：在初始化controller时，**开发者**需要参照上述三种场景，<u>根据需求</u>实现controller的初始化

`./controllers/warpdrivepool_controller.go`（如果是multigroup，则为`./controllers/tos/warpdrivepool_controller.go`）定义了**controller**的<u>核心逻辑</u>，包括：

+ controller**初始化方法**：`SetupWithManager`

+ **调谐**方法：`Reconcile`


其中，Reconcile方法作为最核心的逻辑会在下一节单独细讲，本节主要介绍**SetupWithManager**方法的实现。Controller的启动使用==**Builder模式**==，NewControllerManagerBy 和 For 方法都是给 Builder 传参，最重要的是最后一个方法 Complete。





# 5.Reconcile

`Reconcile`是Controller的主要功能逻辑，所谓**调谐**，就是**将CR的当前状态收敛到期望状态**的**过程**。设计过程中，尽量遵守level-based，而不是edge-based设计原则。这两个名词是电路设计概念：==level-based==触发，指的是接收事件（如中断）并对状态做出反应；==edge-based==触发方式，指的是接收事件并对状态变化做出反应。level-based设计原则要求评估CR的整个状态，而不仅仅是更改的某个状态，更适合在事件可能丢失或重传多次的复杂环境下使用，增强operator的鲁棒性。