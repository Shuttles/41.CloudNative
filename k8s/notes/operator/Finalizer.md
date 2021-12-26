

# 报错

1. 刚写完demo，运行不起来

   ```shell
   2021-12-17T15:16:02.798+0800	INFO	controllers.ImoocPod	ImoocPod.batch.tutorial.kubebuilder.io "demo" is invalid: status.podNames: Invalid value: "null": status.podNames in body must be of type array: "null"	{"imoocpod": "default/demo"}
   ```

   + ==原因==：按这个报错，status.podNames不能为空。

   + ==解决方法==：

     在`api/v1alpha1/imoocpod_types.go`中的`ImoocPodStatus`结构体中的`PodNames`字段的json中加上`omitempty`