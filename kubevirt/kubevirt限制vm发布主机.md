# kubevirt限制vm发布主机

## 设置主机污点及label

```
oc adm taint nodes node52.hisun.com vm=true:NoExecute
oc label node node52.hisun.com vm=true
```

## 设置kubevirt 相关插件及vm发布主机

```
for item in kube-cni-plugins-amd64 kube-multus-amd64 kube-ovs-cni-amd64 kube-sriov-cni-amd64 kube-sriov-device-plugin-amd64
do
	kubectl patch ds/$item -n kube-system -p '{"spec":{"template":{"spec":{"nodeSelector":{"vm":"true"},"tolerations":[{"key":"vm","operator":"Equal","value":"true","effect":"NoExecute"}]}}}}'
done

kubectl patch ds/virt-handler -n kubevirt -p '{"spec":{"template":{"spec":{"nodeSelector":{"vm":"true"},"tolerations":[{"key":"vm","operator":"Equal","value":"true","effect":"NoExecute"}]}}}}'
```

## 发布容忍污点的vm

亲和性相关配置

```
...
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: vm
                operator: In
                values:
                - 'true'
...
```

容忍相关主机配置

```
		...
      hostname: alpine3115-v0270-virt
      tolerations:
        - key: "vm"
          operator: "Equal"
          value: "true"
          effect: "NoExecute"
      networks:
      ...
```

完整yaml

```
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: alpine3115-v0270-virt
spec:
  dataVolumeTemplates:
    - metadata:
        name: alpine3115-v0270-virt-rootdisk
      spec:
        pvc:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
          storageClassName: custom-nas-storage
        source:
          http: 
            url: http://172.26.163.182:8081/packages/kubevirt-image/alpine3115-virt-base-20200403121456.qcow2.gz
      status: {}
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        vm.kubevirt.io/name: alpine3115-v0270-virt
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: vm
                operator: In
                values:
                - 'true'
      domain:
        cpu:
          cores: 1
        devices:
          disks:
            - bootOrder: 1
              disk:
                bus: virtio
              name: rootdisk
            - bootOrder: 3
              cdrom:
                bus: sata
              name: cloudinitdisk
          interfaces:
            - bootOrder: 2
              masquerade: {}
              name: nic0
          rng: {}
        machine:
          type: ''
        resources:
          requests:
            memory: 128M
      hostname: alpine3115-v0270-virt
      tolerations:
        - key: "vm"
          operator: "Equal"
          value: "true"
          effect: "NoExecute"
      networks:
        - name: nic0
          pod: {}
      terminationGracePeriodSeconds: 0
      volumes:
        - dataVolume:
            name: alpine3115-v0270-virt-rootdisk
          name: rootdisk 
        - name: cloudinitdisk
          cloudInitNoCloud:  
            userData: |
              # env
              export SERVER_PORT=5140
              export SERVER_TRACKROOTS=http://x.x.x.x:5145/
              export LEVELDB_OPENPATH=/data/dbfile
              export SECRETKEY_DIR=/data/secretkey
              export SERVER_VERSION=v0.5.3.3
              export PN_DATADIR=/data/pndata
```