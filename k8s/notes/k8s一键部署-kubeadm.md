1. 正因为如此，kubeadm 选择了一种妥协方案：

   > 把 kubelet 直接运行在宿主机上，然后使用容器部署其他的 Kubernetes 组件。