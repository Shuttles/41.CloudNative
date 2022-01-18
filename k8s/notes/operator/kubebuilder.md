



# 常用命令

1. **初始化项目**

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

2. **创建api**

   ```shell
   kubebuilder create api --group batch --version v1 --kind CronJob
   # 当第一次我们为每个组-版本调用这个命令的时候，它将会为新的组-版本创建一个目录。
   
   # 在本案例中，创建了一个对应于batch.tutorial.kubebuilder.io/v1（记得我们在开始时 --domain 的设置吗？) 的 api/v1/ 目录。
   
   # 它也为我们的CronJob Kind 添加了一个文件，api/v1/cronjob_types.go。每次当我们用不同的 kind 去调用这个命令，它将添加一个相应的新文件。
   ```

   

