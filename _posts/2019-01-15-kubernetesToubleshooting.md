---
layout: post
title: Kubernetes troubleshooting  
category: kubernetes
tags: [kubernetes]
---

### I/O timeout when using "kubeadm join"
```
 kubeadm join $master_node:6443 --token ... --discovery-token-ca-cert-hash ......                                  1b9565812597ac42f79bd0aa61990e5182368050171745
.......
[discovery] Failed to request cluster info, will try again: [Get https://$master_node:6443/api/v1/namespaces/kube-public/configmaps/cluster-info: 
dial tcp $master_node:6443: i/o timeout]
```

We tried to disable firewall, add firewall rules, got same message.  
curl https://$master_node:6443  works fine though. 
It turns out that the master node and 2 worker nodes are all VMs created on openStack, add security group rules to allow incoming traffic fix the issue. 

### find which pod takes most memory, in namespace default 
kubectl get pods 
kubectl get pods --output='name'  
kubectl get pods -o=name |  sed "s/^.\{5\}//"      # to get pods name 
kubectl get pods -o=name | awk -F/ '{print $2}'

kubectl get pods -o=name |  sed "s/^.\{5\}//" | kubectl top pod 
kubectl get pods -o=name |  sed "s/^.\{5\}//" | kubectl top pod | sort -k3   # sort by 3rd column 
```
hostnames-85f6f45dc4-2jksc          0m           0Mi
hostnames-85f6f45dc4-jb94j          0m           0Mi
hostnames-85f6f45dc4-rhx9c          0m           0Mi
nginx-deployment-5976b87d54-b6xk2   0m           1Mi
nginx-deployment-5976b87d54-hjlf4   0m           1Mi
nginx-deployment-5976b87d54-mwkx9   0m           1Mi
nginx-deployment-5976b87d54-s6l27   0m           1Mi
node1-s2q5j                         0m           7Mi
NAME                                CPU(cores)   MEMORY(bytes)
```
### 
