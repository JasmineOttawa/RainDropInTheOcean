---
layout: post
title: Kubernetes troubleshooting - Eviction Policy
category: kubernetes
tags: [kubernetes]
---

On Kubenetes cluster, there are some pods got evicted due to imagefs low:  
```
[root@kubernetes-master1-mylab-24729-24-03-2019 ~]# kubectl get nodes
NAME                                               STATUS    ROLES               AGE       VERSION
kubernetes-master1-mylab-24729-24-03-2019   Ready     controlplane,etcd   24d       v1.10.11
kubernetes-node1-mylab-24729-24-03-2019     Ready     worker              24d       v1.10.11
kubernetes-node2-mylab-24729-24-03-2019     Ready     worker              24d       v1.10.11
kubernetes-node3-mylab-24729-24-03-2019     Ready     worker              24d       v1.10.11
kubernetes-node4-mylab-24729-24-03-2019     Ready     worker              24d       v1.10.11
kubernetes-node5-mylab-24729-24-03-2019     Ready     worker              24d       v1.10.11

kubectl get pods --namespace=ncso -o wide  # some running, some in evicted status 
kubectl get pods --namespace=ncso -o wide | grep Running | wc -l   # 45 running 
kubectl get pods --namespace=ncso -o wide | grep Evicted | wc -l   # 894 evicted 
# reason doe evicted pod 
kubectl describe pod ncso-wfm-754c94984c-zt4sh --namespace=ncso
Status:         Failed
Reason:         Evicted
Message:        The node was low on resource: imagefs.
 
[root@kubernetes-node5-mylab-24729-24-03-2019 ~]# df -h /
Filesystem             Size  Used Avail Use% Mounted on
/dev/mapper/vh00-root   50G  4.3G   46G   9% /
[root@kubernetes-node5-mylab-24729-24-03-2019 docker]# cd /var/lib/docker/overlay2

# /var/lib/docker is the imagefs 
[root@kubernetes-node5-mylab-24729-24-03-2019 overlay2]# df -h .
Filesystem                 Size  Used Avail Use% Mounted on
/dev/mapper/docker-docker   69G   31G   39G  45% /var/lib/docker

```
 
# The node was low on resource: imagefs   
https://docs.openshift.com/container-platform/3.5/admin_guide/out_of_resource_handling.html#out-of-resource-eviction-of-pods   
The imagefs is the file system that the container runtime uses for storing images and individual container-writable layers. Eviction thresholds are at 85% full for imagefs. The imagefs file system depends on the runtime and, in the case of Docker, which storage driver you are using.  
# Configure Out Of Resource handling    
https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/#eviction-policy   

There is no way to check the eviction policy info unless hard eviction thresholds are defined and set in kubelet. like:    
--eviction-hard=memory.available<100Mi,nodefs.available<10%,nodefs.inodesFree<5%    




















  

