







1. 一开始在pod中使用volume，要在 Pod 的 YAML 文件里填上 Volumes 字段。，

   但是后来，该成了使用PV 和 PVC对象。

2. StorageClass对象是运维人员创建的，PVC对象是开发人员创建的，PV是K8s根据StorageClass对象创建的