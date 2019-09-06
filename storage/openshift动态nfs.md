# openshift动态nfs

参考：

https://github.com/debianmaster/openshift-examples/tree/master/dynamic-nfs-on-openshift

https://github.com/kubernetes-incubator/external-storage/tree/master/nfs


```
git clone https://github.com/kubernetes-incubator/external-storage.git
cd external-storage/nfs
oc adm policy add-scc-to-user privileged system:serviceaccount:default:nfs-provisioner
oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:default:nfs-provisioner
kubectl create -f deploy/kubernetes/deployment.yaml
oc patch deployment nfs-provisioner -p '{"spec":{"template":{"spec":{"containers":[{"name":"nfs-provisioner","securityContext":{"privileged":true}}]}}}}'
kubectl create -f deploy/kubernetes/class.yaml
oc patch storageclass example-nfs -p '{"metadata":{"annotations":{"storageclass.beta.kubernetes.io/is-default-class":"true"}}}'
## 所有需要挂载nfs的宿主机开放selinux权限，允许pod挂载nfs
setsebool virt_use_nfs on
```

