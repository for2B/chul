---
layout:     post
title:      "使用minio对象存储桶作为pv资源提供给pod挂载"
subtitle:   "Minio实践记录"
date:       2019-09-02
author:     "CHuiL"
header-img: "/img/minio-bg.jpg"
tags:
- minio
---



## minio
MinIO 是一个基于Apache License v2.0开源协议的对象存储服务。它兼容亚马逊S3云存储服务接口，非常适合于存储大容量非结构化的数据，例如图片、视频、日志文件、备份数据和容器/虚拟机镜像等，而一个对象文件可以是任意大小，从几kb到最大5T不等。  

## s3fs-fuse
S3fs是基于FUSE的文件系统，允许Linux和Mac Os X 挂载S3的存储桶在本地文件系统，S3fs能够保持对象原来的格式。关于S3fs的详细介绍，请参见：https://github.com/s3fs-fuse/s3fs-fuse  

使用s3fs-fuse便可以使原来的应用不需进行修改便可以将数据保存到s3云存储服务上，同理也可以保存到我们的minio对象存储上。  

## 使用s3fs-fuse来将minio存储桶挂载到本地文件系统
### 准备minio
在k8s上运行minio pod

```
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: minio-deployment
spec:
  selector:
    matchLabels:
      app: minio
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio:latest
        args:
        - server
        - /storage
        env:
        - name: MINIO_ACCESS_KEY
          value: "minio"
        - name: MINIO_SECRET_KEY
          value: "minio123"
        ports:
        - containerPort: 9000
          hostPort: 9000
        volumeMounts:
        - name: storage 
          mountPath: "/storage"  #将minio的数据挂载到本地系统
      volumes:
        - name: storage
          hostPath:
            path: /data/minio/minio-pod  
            
---
apiVersion: v1
kind: Service
metadata:
  name: minio-service
spec:
  type: NodePort
  ports:
    - port: 9000
      targetPort: 9000
      nodePort: 30900
      protocol: TCP
  selector:
    app: minio

```
我们运行起来一个minio对象存储，并将他的数据保存到本地主机的/data/minio/minio-pod目录下；


### 使用s3fs-fuse来将minio对象存储桶挂载都某个目录下
安装和使用s3fs-fuse可参考官方教程;  
这里建议直接下载最新版本自己安装，否则使用apt-get获取到的版本在后续可能会出现一些问题。

首先准备好passwd_file,将minio的access_key和secret_access_key写入到passwd-s3fs文件中；
```
echo ACCESS_KEY_ID:SECRET_ACCESS_KEY > ${HOME}/.passwd-s3fs
chmod 600 ${HOME}/.passwd-s3fs
```

挂载的命令  
`sudo s3fs {{BUCKERNAME}} {{MOUNTDIR}} -o passwd_file={{PASSWD_FILE_DIR}} -o url=http://{{minio.service}}:9000 -o use_path_request_style `  

例如  

`sudo s3fs test ./minio-mount -o passwd_file=.passwd-s3fs -o url=http://172.31.11.34:9000 -o use_path_request_style `  

这里我们将./minio-mount目录挂载到test这个桶下，并读取passwd_file文件，url写入我们minio服务的地址，并且需要带上use_path_request_style这个参数；

配置成功后，输入mount验证；
```
mount | grep s3fs
fuse.s3fs (rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other)
```

如果没有显示fuse.s3fs,可以运行命令时加-f -d 来输出信息debud

尝试在./minio-mount中创建文件，创建的文件会存储到minio的test桶中；  

到这里我们其实便可以将pod挂载到本地的./minio-mount目录，来将pod数据存储到minio对象存储桶中

## 使用IBM Cloud Object Storage plug-in  
IBM Cloud Object Storage插件是一个Kubernetes卷插件，使Kubernetes pod可以访问IBM Cloud Object Storage存储桶，也可以用来访问我们自己搭建的minio存储桶；  
具体的介绍和安装使用的方法可以看官方教程  [ibmcloud-object-storage-plugin](https://github.com/IBM/ibmcloud-object-storage-plugin)  

ibmc-s3fs文件下载 https://github.com/for2B/ibmcloud-object-storage-plugin/releases/download/untagged-cc0c76dffba4033abfc0/ibmc-s3fs 
 
ibmcloud镜像可以使用下面的镜像
`ish2b/ibmcloud-object-storage-plugin:latest`    


1.复制ibmc-s3fs文件到k8s插件目录,（下面的命令中ibmc-s3fs文件已经存放在/tmp目录下）    

```
$ sudo mkdir -p /usr/libexec/kubernetes/kubelet-plugins/volume/exec/ibm~ibmc-s3fs
$ sudo cp /tmp/ibmc-s3fs /usr/libexec/kubernetes/kubelet-plugins/volume/exec/ibm~ibmc-s3fs
$ sudo chmod +x /usr/libexec/kubernetes/kubelet-plugins/volume/exec/ibm~ibmc-s3fs/ibmc-s3fs
$ sudo systemctl restart kubelet
```

2.创建provisioner.

```
kubectl create -f https://raw.githubusercontent.com/IBM/ibmcloud-object-storage-plugin/master/deploy/provisioner-sa.yaml

kubectl create -f https://raw.githubusercontent.com/IBM/ibmcloud-object-storage-plugin/master/deploy/provisioner.yaml
``` 

3.创建storage class

最后的iam-endpoint地址配置写上我们的minio服务地址，字段的详细含义[在 IBM Cloud Object Storage 上存储数据](https://cloud.ibm.com/docs/containers?topic=containers-object_storage#configure_cos)

```
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: ibmc-s3fs-standard
provisioner: ibm.io/ibmc-s3fs
parameters:
  ibm.io/chunk-size-mb: "10"
  ibm.io/parallel-count: "5"
  ibm.io/multireq-max: "20"
  ibm.io/tls-cipher-suite: "AES"
  ibm.io/stat-cache-size: "100000"
  ibm.io/debug-level: "warn"
  ibm.io/curl-debug: "false"
  ibm.io/kernel-cache: "true"
  ibm.io/s3fs-fuse-retry-count: "5"
  ibm.io/iam-endpoint: "http://172.31.11.34:9000"
```

验证插件和stroageclass是否安装成功
```
$ kubectl get pods -n kube-system | grep object-storage
ibmcloud-object-storage-plugin-6d8464c657-5zpfh   1/1     Running            0          3d

$ kubectl get storageclass |grep s3
ibmc-s3fs-standard   ibm.io/ibmc-s3fs   2d22h

```

4 创建Secret,这里需要将minio的access-key secret-key进行base64编码；
```
apiVersion: v1
kind: Secret
type: ibm/ibmc-s3fs
metadata:
  name: minio-secret
data:
  access-key: bWluaW8=
  secret-key: bWluaW8xMjM=
```

5.创建pvc
这里endpoint也配置上我们的minio服务器地址，并且配置桶名，在桶不存在时自动创建；注意secret 和pvc需要在一个命名空间下
```
 kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ibmc-s3fs-pvc
  annotations:
    volume.beta.kuenetes.io/storage-class: "ibmc-s3fs-standard"
    ibm.io/auto-create-bucket: "true"
    ibm.io/auto-delete-bucket: "false"
    ibm.io/bucket: "ibmc-s3fs-bucket"
    ibm.io/object-path: ""
    ibm.io/endpoint: "http://172.31.11.34:9000"
    ibm.io/region: "us-standard"
    ibm.io/secret-name: "minio-secret"
    ibm.io/stat-cache-expire-seconds: ""
spec:
  storageClassName: ibmc-s3fs-standard
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 8Gi

```

6.创建测试pod
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: s3fs-test-pod
  namespace: <NAMESPACE_NAME>
spec:
  containers:
  - name: s3fs-test-container
    image: anaudiyal/infinite-loop
    volumeMounts:
    - mountPath: "/mnt/s3fs"
      name: s3fs-test-volume
  volumes:
  - name: s3fs-test-volume
    persistentVolumeClaim:
      claimName: s3fs-test-pvc
EOF
```

验证是否挂载成功


```
$ kubectl get pod |grep s3fs
s3fs-test-pod                             1/1     Running   0          2d21h

$ kubectl exec -it s3fs-test-pod -n <NAMESPACE_NAME> bash
root@s3fs-test-pod:/#
root@s3fs-test-pod:/# df -Th | grep s3
s3fs           fuse.s3fs  256T     0  256T   0% /mnt/s3fs

root@s3fs-test-pod:/# cd /mnt/s3fs/
root@s3fs-test-pod:/mnt/s3fs# ls
root@s3fs-test-pod:/mnt/s3fs#

root@s3fs-test-pod:/mnt/s3fs# echo "IBM Cloud Object Storage plug-in" > sample.txt
root@s3fs-test-pod:/mnt/s3fs# ls
sample.txt
root@s3fs-test-pod:/mnt/s3fs# cat sample.txt
IBM Cloud Object Storage plug-in
root@s3fs-test-pod:/mnt/s3fs#
```

至此使用IBM Cloud Object Storage plug-in来将minio作为pv资源挂载就完成了，我们只需要按照需求创建好pvc，便可以由strageclass自动创建pv；

## 卸载
```
$ kubectl delete deployment ibmcloud-object-storage-plugin -n kube-system
$ kubectl delete clusterRoleBinding ibmcloud-object-storage-plugin ibmcloud-object-storage-secret-reader
$ kubectl delete clusterRole ibmcloud-object-storage-plugin ibmcloud-object-storage-secret-reader
$ kubectl delete sa ibmcloud-object-storage-plugin -n kube-system
$ kubectl delete sc ibmc-s3fs-standard
```
只需将刚才创建的sc,sa deploy,pod等删除即可。

#### 整体架构图
 
![image](/chuil/img/minio/2019-09-02-1.png)
 
## 参考  

[ibmcloud-object-storage-plugin](https://github.com/IBM/ibmcloud-object-storage-plugin)     

[在 IBM Cloud Object Storage 上存储数据](https://cloud.ibm.com/docs/containers?topic=containers-object_storage)  

[s3fs-fuse](https://github.com/s3fs-fuse/s3fs-fuse)