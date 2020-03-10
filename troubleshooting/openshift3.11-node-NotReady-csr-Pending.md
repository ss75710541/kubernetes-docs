# 解决openshift3.11 node NotReady csr Pending 

查看节点状态

```
# oc get nodes
NAME                     STATUS     ROLES           AGE       VERSION
infra196.hisun.com    NotReady   compute,infra   1y        v1.11.0+d4cacc0
```

查看节点证书列表

```
# oc get csr
NAME                                                   AGE       REQUESTOR                                                 CONDITION
node-csr-AWiUeeSSCGyQt1RMoc-ij5A6tk06zbcsCwIaY3Bw_4M   21s       system:serviceaccount:openshift-infra:node-bootstrapper   Pending
```

查看Pending csr

```
# oc describe csr node-csr-AWiUeeSSCGyQt1RMoc-ij5A6tk06zbcsCwIaY3Bw_4M
Name:               node-csr-AWiUeeSSCGyQt1RMoc-ij5A6tk06zbcsCwIaY3Bw_4M
Labels:             <none>
Annotations:        <none>
CreationTimestamp:  Tue, 10 Mar 2020 18:43:00 +0800
Requesting User:    system:serviceaccount:openshift-infra:node-bootstrapper
Status:             Pending
Subject:
         Common Name:    system:node:infra196.hisun.com
         Serial Number:
         Organization:   system:nodes
Events:  <none>
```

审批通过 csr

```
# oc adm certificate approve node-csr-AWiUeeSSCGyQt1RMoc-ij5A6tk06zbcsCwIaY3Bw_4M
certificatesigningrequest.certificates.k8s.io/node-csr-AWiUeeSSCGyQt1RMoc-ij5A6tk06zbcsCwIaY3Bw_4M approved
```

查看节点状态

```
# oc get nodes
NAME                     STATUS    ROLES           AGE       VERSION
infra196.hisun.com    Ready     compute,infra   1y        v1.11.0+d4cacc0
```