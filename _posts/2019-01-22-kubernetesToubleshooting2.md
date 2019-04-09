---
layout: post
title: Kubernetes troubleshooting - 2  
category: kubernetes
tags: [kubernetes]
---

###  list all pods running behind a service 
###  list the which pod has highest CPU 
###  list the node which is ready, and not has effect=unschedule 

###  start apiserver if it's down 
###  etcd snapshot 
###  systemd kubelet program 
###  add a node to cluster 
###  daemonset by default goes to worker node only, what about if you want to put a pod on master node as well 


### kubernetes dashboard access 
Issue: A dashboard service was added into a kubercluster, per https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml
kubectl proxy
Kubectl will make Dashboard available at http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
The UI can only be accessed from the machine where the command is executed. 
```
After running above commands, the dashboard is not available even on localhost 

Investigation: 
step1, verify if the pod listening on port 8443. yes pod:port = 10.32.0.4:8443, service_IP:port=10.111.227.158:443
```
[root@centos-1 ~]# kubectl get pods --namespace=kube-system | grep dashboard
kubernetes-dashboard-5f7b999d65-pv9mx        1/1     Running            0          17h
[root@centos-1 ~]# kubectl get pods kubernetes-dashboard-5f7b999d65-pv9mx --namespace=kube-system --template='{{(index (index .spec.containers 0).ports 0).containerPort}}{{"\n"}}'
8443
[root@centos-1 ~]# kubectl describe service kubernetes-dashboard --namespace=kube-system
Name:              kubernetes-dashboard
Namespace:         kube-system
Labels:            k8s-app=kubernetes-dashboard
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"k8s-app":"kubernetes-dashboard"},"name":"kubernetes-dashboard"...
Selector:          k8s-app=kubernetes-dashboard
Type:              ClusterIP
IP:                10.111.227.158
Port:              <unset>  443/TCP
TargetPort:        8443/TCP
Endpoints:         10.32.0.4:8443
Session Affinity:  None
Events:            <none>
```

step2, port forward,1st one is the port on node, 2nd one is the service port. this way does not work 
```
[root@centos-1 ~]#  kubectl port-forward svc/kubernetes-dashboard 8443:8443 --namespace=kube-system
error: Service kubernetes-dashboard does not have a service port 8443
[root@centos-1 ~]# kubectl port-forward svc/kubernetes-dashboard 8443:443 --namespace=kube-system
Forwarding from 127.0.0.1:8443 -> 8443
Forwarding from [::1]:8443 -> 8443
```
curl http://127.0.0.1:8443 does not work

step3, kube proxy, this one works 
get server IP 
```
kubectl config view
 server: https://10.12.5.68:6443
kubectl proxy --port=9999 --address='10.12.5.68' --accept-hosts="^*$"
```
Put following in browser http://10.12.5.68:9999/ui
it works, while not much help, it displays information, not the interactive GUI 

step4, later find this k8s cluster is built up using rancher, using rancher GUI instead. 








