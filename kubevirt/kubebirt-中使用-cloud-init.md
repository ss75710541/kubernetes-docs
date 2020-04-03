# kubebirt 中使用 cloud-init

## 发布 fedora 测试 cloud-init

fedora-virt.yaml

```
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: fedora-v0270-virt
spec:
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        vm.kubevirt.io/name: fedora-v0270-virt
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
              name: containerdisk
            - bootOrder: 3
              disk:
                bus: virtio
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
            memory: 512M
      hostname: fedora-v0270-virt
      networks:
        - name: nic0
          pod: {}
      terminationGracePeriodSeconds: 0
      volumes:
        - name: containerdisk
          containerDisk:
            image: kubevirt/fedora-cloud-container-disk-demo:latest
        - name: cloudinitdisk
          cloudInitNoCloud:  
            userData: |
              #cloud-config
              ssh_pwauth: yes
              chpasswd:
                list: |
                    root:123456
              users:
                - name: root
                  ssh-authorized-keys: >-
                    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDOV3wVuXAIqkgrzUtWeQEjXLbXSHyJP+8ZewUFnyDrTkiW06haU/GcN0+E2UqRHxalaYQile7lBu2SpW2QryjYntGlvefjPSnmE206yI7N07nCPrFWmu4UUAEAEy+LFAjJ+NUmiqBJYsE1aR1rgWskRljzmmYAkB3L672bIsID05rmckJs1+oIvbxxXS86WVjGERTDHH6pCHDHYeb/yyPbM8LiqLBUNSJDNOuAtIRynNarkgZhj+evIbrbpnxhPzD+ZoYoSsweXr8vn7g8vrosiDrI6BPJIvULYp821Zthb5CDR6DYsTDzaTQgBenmUm9eUp0nZ+WSwJW4QvMB9FJr
```

## 发 alpine 测试 cloud-init

注：fedora服务中是包含了cloud-init服务的，alpine 默认是不支持cloud-init的，所以我们需要自己加载cloud-init中的user-data 数据，为了`/dev/编号`的名称操作简便，clountinitdisk 我们的挂载为cdrom模式，默认在vm 中是 /dev/sr0, 使用`mount -t iso9660 /dev/sr0 /mnt` 就可以看到从/mnt 看到 cloud-init 数据了。可以写一些自定义的脚本加载指定内容。

alpine-virt.yaml

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
      hostname: alpine310-v0270-virt
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