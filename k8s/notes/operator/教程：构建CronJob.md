

参考资料：

zh：https://cloudnative.to/kubebuilder/cronjob-tutorial/cronjob-tutorial.html

en：https://go.kubebuilder.io/cronjob-tutorial/cronjob-tutorial.html



# 0.前言

1. 本教程将会带你了解 Kubebuilder 的（几乎）全部复杂功能，从简单的功能开始，到全部功能。
2. 让我们假装厌倦了 Kubernetes 中的 CronJob 控制器的非 Kubebuilder 实现繁重的维护工作（当然，这有点小题大做），我们想用 KubeBuilder 来重写它。
3. **CronJob** 控制器的工作是定期在 Kubernetes 集群上运行一次性任务。它是以 **Job** 控制器为基础实现的，而 **Job** 控制器的任务是运行一次性的任务，确保它们完成。
4. 与其试图一开始解决重写 Job 控制器的问题，我们先看看如何与外部类型进行交互。



# 1.基本项目中有什么

1. **搭建项目**

   ```shell
   mkdir cronjbo
   cd cronjob
   
   # 我们将使用 tutorial.kubebuilder.io 域，
   # 所以所有的 API 组将是<group>.tutorial.kubebuilder.io.
   # --repo好像是用来go mod init的？
   kubebuilder init --domain tutorial.kubebuilder.io --repo tutorial.kubebuilder.io/cronjob
   
   # 官方文档这么写的：
   # 如果您的项目在 [ GOPATH][GOPATH-golang-docs] 中初始化，则隐式调用go mod init将为您插入模块路径。否则--repo=<module path>必须设置
   ```

   之后可以获得：

2. 构建项目的**基础设施**

   + `go.mod`
   + `Makefile`
   + `PROJECT`用于搭建新组件的 Kubebuilder 元数据

3. **启动配置**

   我们还在 [`config/`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config) 目录下获得**启动配置**。现在，它只包含 在集群上启动控制器所需的[Kustomize](https://sigs.k8s.io/kustomize) YAML 定义，但是一旦我们开始编写控制器，它还将保存我们的 CustomResourceDefinitions、RBAC 配置和 WebhookConfigurations。

   [`config/default`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config/default)包含一个[Kustomize 基础](https://github.com/kubernetes-sigs/kubebuilder/blob/master/docs/book/src/cronjob-tutorial/testdata/project/config/default/kustomization.yaml)，用于在标准配置中启动控制器。

   每个其他目录都包含不同的配置，重构为自己的基础：

   - [`config/manager`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config/manager): 将你的控制器作为集群中的 pod 启动
   - [`config/rbac`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config/rbac)：在自己的服务帐户下运行控制器所需的权限

4. **入口点**

   最后但同样重要的是，Kubebuilder 搭建了我们项目的基本入口点：`main.go`. 让我们接下来看看...





# 2.main.go

1. 我们的 main 文件最开始是 import 一些基本库，尤其是：

   - 核心的 [controller-runtime](https://pkg.go.dev/sigs.k8s.io/controller-runtime?tab=doc) 库
   - 默认的控制器运行时**日志**库-- Zap

2. 每一组控制器都需要一个[==*Scheme*==](https://book.kubebuilder.io/cronjob-tutorial/gvks.html#err-but-whats-that-scheme-thing)，它提供了 **Kinds 和相应的 Go 类型之间**的**映射**

   ```go
   var (
       scheme   = runtime.NewScheme()
       setupLog = ctrl.Log.WithName("setup")
   )
   ```

3. main.go的**核心逻辑**：

   + 通过flag库解析入参

   + **实例化**一个`manager`，其作用：

     + 它记录着我们所有`controller`的运行情况
     + setting up **shared caches** and **clients** to the API server

     注意，我们把**Scheme** 的信息告诉了 manager

   + **运行 manager**，它反过来运行我们所有的**controllers和 webhooks**

     manager 状态被设置为 Running，直到它收到一个优雅停机 (graceful shutdown) 信号。这样一来，当我们在 Kubernetes 上运行时，我们就可以优雅地停止 pod。

4. 注意：Manager 可以通过以下方式<u>限制控制器可以监听资源的命名空间</u>。

   ```go
       mgr, err = ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
           Scheme:             scheme,
           Namespace:          namespace,
           MetricsBindAddress: metricsAddr,
       })
   ```

   上面的例子将把你的项目改成**只监听单一的命名空间**。在这种情况下，建议通过将默认的 ClusterRole 和 ClusterRoleBinding 分别替换为 Role 和 RoleBinding 来限制所提供给这个命名空间的授权。



# 3.GVK GVR



**Group Version**

Kubernetes 中的 **API Group**简单来说就是**相关功能的集合**。每个组都有一个或多个**Version**，顾名思义，它允许我们随着时间的推移改变 **API** 的职责。



**Kind resources**

1. 每个 API 组-版本包含一个或多个 API **类型**，我们称之为 **Kinds**。虽然一个 Kind 可以在不同版本之间改变表单内容，但每个表单必须能够以某种方式存储其他表单的所有数据（我们可以将数据存储在字段中，或者在注释中）。 这意味着，<u>使用旧的 API 版本不会导致新的数据丢失或损坏</u>。更多 API 信息，请参阅 [Kubernetes API 指南](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md)。
2. 你也会偶尔听到提到 **resources**。 resources（资源） 只是 API 中的一个 Kind 的使用方式。通常情况下，Kind 和 resources 之间有一个一对一的映射。 例如，`pods` **resources**对应于 `Pod` **Kind**。
3. 注意：resources 总是用小写，按照惯例是 Kind 的小写形式。



==**为什么要创建api？**==

让我们考虑一下经典场景，目标是让应用程序及其数据库在 Kubernetes 平台上运行。然后，**一个 CRD 可以代表 App，另一个可以代表 DB**。通过一个 CRD 来描述 App，另一个 CRD 来描述 DB，我们不会损害**封装、单一职责原则和内聚**等概念。破坏这些概念可能会导致意想不到的副作用，例如难以扩展、重用或维护，仅举几例。

通过这种方式，我们可以创建应用程序 CRD，该应用程序 CRD 将具有其控制器，并负责创建包含应用程序的部署和创建服务以访问它等。类似地，我们可以创建一个 CRD 来表示数据库，并部署一个管理数据库实例的控制器。



==**Scheme是什么？**==

1. 我们之前看到的 `Scheme` 是一种**追踪 Go Type 的方法**，它对应于给定的 GVK（不要被它吓倒 [godocs](https://pkg.go.dev/k8s.io/apimachinery/pkg/runtime#Scheme)）。



# 4.Create API



```shell
kubebuilder create api --group batch --version v1 --kind CronJob
```

1. 当**第一次**我们为每个组-版本调用这个命令的时候，它将会为新的组-版本**创建一个目录**。
2. 在本案例中，创建了一个对应于`batch.tutorial.kubebuilder.io/v1`（记得我们在开始时 [`--domain`](https://cloudnative.to/kubebuilder/cronjob-tutorial/cronjob-tutorial.html#scaffolding-out-our-project) 的设置吗？) 的 [`api/v1/`](https://sigs.k8s.io/kubebuilder/docs/book/src/cronjob-tutorial/testdata/project/api/v1) 目录。
3. 它也为我们的`CronJob` Kind 添加了一个文件，`api/v1/cronjob_types.go`。每次当我们用不同的 kind 去调用这个命令，它将添加一个相应的新文件。





1. 下一步，我们为`Kind`的 `Spec`和 `Status` 定义type。

2. Kubernetes 通过**reconcile**所需状态 ( `Spec`) 与 实际集群状态（其他对象的 `Status`）和外部状态 来发挥作用，然后记录它观察到的内容 ( `Status`)。

   === status，它表示实际看到的状态。它包含了我们希望用户或其他控制器能够轻松获得的任何信息。==

   因此，每个*功能*对象都包含**Spec**和**Status**。很少的类型，像 `ConfigMap` 不需要遵从这个模式，因为它们不编码期待的状态， 但是大部分类型需要做这一步。

3. 下一步，我们定义与实际Kinds相对应的types--`CronJob` 和 `CronJobList` 。 `CronJob` 是一个**根类型**, 它描述了 `CronJob` Kind。像所有 Kubernetes 对象那样，它包含 `TypeMeta` (描述了API版本和种类)，也包含其中有像name, namespace和labels的东西的 `ObjectMeta` 。

   `CronJobList` 是多个 `CronJob` 的容器。它是批量操作中使用的Kind，像 `LIST `操作。

   通常情况下，我们从不修改任何一个 -- <u>所有修改都要到 Spec 或者 Status</u> 。

4. 那个小小的 `+kubebuilder:object:root` 注释被称为**标记**(marker)。我们将会看到更多的它们，但要知道它们其实就是**额外的元数据**， 告诉[controller-tools](https://github.com/kubernetes-sigs/controller-tools)(我们的代码和YAML生成器)**额外的信息**。 

   这个特定的标签告诉 `object `生成器这个go type表示一个Kind。然后，`object` 生成器为我们生成 [runtime.Object](https://pkg.go.dev/k8s.io/apimachinery/pkg/runtime?tab=doc#Object)**接口的实现**。(所有表示Kind的go type一定要实现)

   

6. Finally, we add the Go types to the API group. This allows us to add the types in this API group to any [Scheme](https://pkg.go.dev/k8s.io/apimachinery/pkg/runtime?tab=doc#Scheme).

   ```go
   func init() {
       SchemeBuilder.Register(&CronJob{}, &CronJobList{})
   }
   ```



# 5.Design API



1. 在 Kubernetes 中，我们对如何设计 API 有一些原则。

   + 也就是说，所有序列化的字段**必须**是 `驼峰式` ，所以我们使用的 **JSON** 标签需要遵循该格式。我们也可以使用**`omitempty`** 标签来标记一个字段在空的时候应该在序列化的时候省略。

   + 字段**可以使用大多数的基本类型**。数字是个例外：出于 API 兼容性的目的，我们只允许三种数字类型。对于整数，需要使用 `int32` 和 `int64` 类型；对于小数，使用 `resource.Quantity` 类型。

     还有一个我们使用的特殊类型：`metav1.Time`。 它有一个稳定的、可移植的序列化格式的功能，其他与 `time.Time` 相同。

2. 最后，CronJob 和 CronJobList 直接使用模板生成的即可。如前所述，我们不需要改变这个，除了标记我们想要一个**有状态subresource**，**这样我们的行为就像内置的 kubernetes 类型。**==不懂==



# 剩下文件的作用

1. 如果你在 [`api/v1/`](https://sigs.k8s.io/kubebuilder/docs/book/src/cronjob-tutorial/testdata/project/api/v1) 目录下看到了其他文件， 你可能会注意到除了 `cronjob_types.go` 这个文件外，还有两个文件：`groupversion_info.go` 和 `zz_generated.deepcopy.go`。

   虽然这些文件都不需要编辑(前者保持原样，而后者是自动生成的)，但是如果知道这些文件的内容，那么将是非常有用的。

2. **groupversion_info.go**

   + 包含了关于 group-version 的一些**元数据**:

   + 首先，我们有一些*包级别的*标记，表示这个包中有 Kubernetes 对象，并且这个包代表组 `batch.tutorial.kubebuilder.io`。

   + 然后，我们有一些常见且常用的变量来帮助我们设置我们的 **Scheme** 。因为我们需要在这个包的 controller 中用到所有的go types， 用一个方便的方法**给其他 `Scheme` 来添加所有的类型**（==不懂==），是非常有用的(而且也是一种惯例)。SchemeBuilder 能够帮助我们轻松的实现这个事情。

     ```go
     var (
         // GroupVersion is group version used to register these objects
         GroupVersion = schema.GroupVersion{Group: "batch.tutorial.kubebuilder.io", Version: "v1"}
     
         // SchemeBuilder is used to add go types to the GroupVersionKind scheme
         SchemeBuilder = &scheme.Builder{GroupVersion: GroupVersion}
     
         // AddToScheme adds the types in this group-version to the given scheme.
         AddToScheme = SchemeBuilder.AddToScheme
     )
     ```

3. **zz_generated.deepcopy.go**

   + `zz_generated.deepcopy.go` 包含了前述 `runtime.Object` 接口的自动实现，这些实现标记了 代表 `Kinds` 的所有根类型。
   + `runtime.Object` 接口的核心是一个深拷贝方法，即 `DeepCopyObject`。
   + controller-tools 中的 `object` 生成器也能够为每一个根类型以及其子类型生成另外两个易用的方法：`DeepCopy` 和 `DeepCopyInto`。





# 6.Controller中有什么

1. 控制器是 Kubernetes 的核心，也是任何 operator 的**核心**。

2. 控制器的工作是**确保对于任何给定的对象**，<u>世界的实际状态</u>（包括集群状态，以及潜在的外部状态，如 Kubelet 的运行容器或云提供商的负载均衡器）<u>与对象中的期望状态相匹配</u>。每个控制器专注于一个根 Kind，但可能会与其他 Kind 交互。

   我们把这个过程称为 **reconciling**。

3. 在 controller-runtime 中，为特定种类实现 reconciling 的逻辑被称为 [*Reconciler*](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/reconcile)。 <u>==Reconciler 接受一个对象的名称，并返回我们是否需要再次尝试==</u>（例如在错误或周期性控制器的情况下，如 HorizontalPodAutoscaler）。



**对controller.go的讲解**

1. `Reconcile` 实际上是对**单个对象**进行调谐。我们的 Request 只是有一个名字（namespacedname）（<u>说是request，但实际上就指定了object，没有指定操作</u>），但我们可以使用 client 从**cache**中获取这个对象。
2. 我们返回一个空的result，没有错误，这就向 controller-runtime 表明我们已经成功地对这个对象进行了调谐，在有一些变化之前**不需要**再尝试调谐。
3. 大多数控制器需要一个**日志句柄**（logging handle）和一个**上下文**（context），所以我们**在 Reconcile 中将他们初始化**。
4. **上下文Context**:
   + 上下文是用来**允许取消请求的**，也或者是**实现 tracing 等功能**。它是所有 client 方法的第一个参数。`Background` 上下文只是一个**基本**的上下文，没有任何额外的数据或超时时间限制。
5. **日志**：
   + controller-runtime通过一个名为logr的库使用**结构化的日志记录**。日志记录的**工作原理**是<u>将kv对附加到静态消息中</u>。我们可以在我们的reconcile方法的顶部预先分配一些kv对，让这些kv对附加到这个reconciler的所有日志行。
6. `SetupWithManager(ctrl.Manager)`
   + 最后，我们将 Reconcile 添加到 manager 中，这样<u>当 manager 启动时它就会被启动</u>。





# 7.实现一个controller

1. 牢记，**status** 值应该是从**实际的运行状态**中**实时获取**。









## main的修改？



1. `init()`中

   ```go
   func init() {
       utilruntime.Must(clientgoscheme.AddToScheme(scheme))
   
       utilruntime.Must(batchv1.AddToScheme(scheme))
       // +kubebuilder:scaffold:scheme
   }
   ```

   

   +  kubebuilder 已经添加了一组新的 API 包（`batchv1`）到 scheme。这意味着**可以在我们的控制器中使用这些对象**。
   + 如果我们要使用任何其他的 CRD，我们必须用相同的方法添加它们的 scheme。内置的类型例如 Job 就添加了它自己的 scheme -- `clientgoscheme`。

2. `main()`中

   + kubebuilder 已经添加了一个**阻塞调用**我们的 CornJob 控制器的 `SetupWithManager` 方法。







# 疑问：

1. subresource是什么？

   在设计api中，有一段话：

   > 最后，CronJob 和 CronJobList 直接使用模板生成的即可。如前所述，我们不需要改变这个，除了标记我们想要一个有状态子资源，这样我们的行为就像内置的 kubernetes 类型。

2. 剩下文件的作用中有不懂的

3. Reconcile的参数request到底代表什么，为什么Get要传入req.NamespacedName？