* 1 创建 dinghao-ns 命名空间
> kubectl create -f namespace-dinghao.yaml

* 2 配置账号权限及角色绑定
> kubectl apply -f nfs-client-provisioner-role.yaml  

* 3 安裝 NFS Client provisioner
> kubectl apply -f nfs-client-provisioner.yaml

* 4 部署storageclass
> kubectl apply -f storageclass.yaml

* 5 部署pvc
> kubectl apply -f PersistentVolumeClaim.yaml

* 6 测试
> kubectl apply -f test-pod.yaml

* 检查部署
```
kubectl get pod,pvc,pv
kubectl get pod,sc,deploy -n dinghao-ns

```
