```
动态pvc,mysql 部署测试
1　创建namespace
2  设定 Service Account 的权限
3  安裝 NFS Client provisioner
4  创建 StorageClass
5  创建 PersistentVolumeClaim
6  mysql 配置文件 my.ini, 将 My.ini 挂载到 Configmap
7  将 MySql 的密码以 Secret 格式保存
8  部署测试
```

* 1 创建namespace
```
cat namespace.yml 
apiVersion: v1
kind: Namespace
metadata:
  name: dinghao-ns
  labels:
    name: dinghao-ns
    part-of: dinghao-ns

```

* 2 设定 Service Account 的权限
```
cat nfs-client-provisioner-role.yaml 

apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io

---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io

```
* 3 安裝 NFS Client provisioner
```
cat nfs-client-provisioner.yaml 
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: my-nfs-provisioner
            - name: NFS_SERVER
              value: 172.16.3.24
            - name: NFS_PATH
              value: /data
      volumes:
        - name: nfs-client-root
          nfs:
            server: 172.16.3.24
            path: /data

```

* 4 创建 StorageClass
```
cat namespace.yml 
apiVersion: v1
kind: Namespace
metadata:
  name: dinghao-ns
  labels:
    name: dinghao-ns
    part-of: dinghao-ns

```
* 5  创建 PersistentVolumeClaim
```
cat PersistentVolumeClaim.yaml 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysqlclaim
  namespace: dinghao-ns
  annotations:
    volume.beta.kubernetes.io/storage-class: "my-nfs-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi

```
* 6 mysql 配置文件 my.ini, 将 My.ini 挂载到 Configmap
```
kubectl create configmap mysqlini --from-file=my.ini --namespace=dinghao-ns
configmap/mysqlini created

sed -e '/^#/d;/^$/d' my.ini 
[client]
[mysqld]
default-time-zone = '+8:00'
back_log = 600
max_connections = 1000
max_connect_errors = 6000
open_files_limit = 65535
table_open_cache = 128
max_allowed_packet = 256M
max_heap_table_size = 8M
tmp_table_size = 16M
read_buffer_size = 2M
read_rnd_buffer_size = 8M
sort_buffer_size = 8M
join_buffer_size = 8M
thread_cache_size = 8
query_cache_size = 8M
query_cache_limit = 2M
key_buffer_size = 4M
ft_min_word_len = 4
transaction_isolation = REPEATABLE-READ
binlog_format = row
performance_schema = 0
explicit_defaults_for_timestamp=true
default-storage-engine = InnoDB #默认存储引擎
innodb_file_per_table = 1
innodb_open_files = 500
innodb_buffer_pool_size = 64M
innodb_write_io_threads = 4
innodb_read_io_threads = 4
innodb_thread_concurrency = 0
innodb_purge_threads = 1
innodb_flush_log_at_trx_commit = 2
innodb_log_buffer_size = 2M
innodb_log_file_size = 2G
innodb_log_files_in_group = 3
innodb_max_dirty_pages_pct = 90
innodb_lock_wait_timeout = 120
bulk_insert_buffer_size = 8M
myisam_sort_buffer_size = 8M
myisam_max_sort_file_size = 10G
myisam_repair_threads = 1
interactive_timeout = 28800
wait_timeout = 28800
character-set-server=utf8
collation-server=utf8_bin
transaction-isolation=READ-COMMITTED

```
* 7 将 MySql 的密码以 Secret 格式保存
```
kubectl create secret generic mysql-secret --from-literal=password=123456 --namespace=dinghao-ns
```
* 8  部署测试
```
kubectl get pv,pvc,ns,sc
kubectl get pod -o wide
kubectl describe pod nfs-client-provisioner-756d984cb5-fps9s

kubectl get pod -n dinghao-ns -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
mshk-mysql-85b94dd4b8-99bm2   1/1     Running   0          21h   10.244.1.46   k8s-node1   <none>           <none>

```

* ９　遇到的问题
```
执行如下目录，解决提示报错问题
yum -y install nfs-utils rpcbind

kubectl describe pod nfs-client-provisioner-756d984cb5-fps9s

Mounting command: systemd-run
Mounting arguments: --description=Kubernetes transient mount for /var/lib/kubelet/pods/b85fa0bb-9660-11e9-9e70-52540027b914/volumes/kubernetes.io~nfs/nfs-client-root --scope -- mount -t nfs 172.16.3.24:/data /var/lib/kubelet/pods/b85fa0bb-9660-11e9-9e70-52540027b914/volumes/kubernetes.io~nfs/nfs-client-root
Output: Running scope as unit run-16696.scope.
mount: 文件系统类型错误、选项错误、172.16.3.24:/data 上有坏超级块、
       缺少代码页或助手程序，或其他错误
       (对某些文件系统(如 nfs、cifs) 您可能需要
       一款 /sbin/mount.<类型> 助手程序)

       有些情况下在 syslog 中可以找到一些有用信息- 请尝试
       dmesg | tail  这样的命令看看。


mount: 文件系统类型错误、选项错误、172.16.3.24:/data 上有坏超级块、
       缺少代码页或助手程序，或其他错误
       (对某些文件系统(如 nfs、cifs) 您可能需要
```
