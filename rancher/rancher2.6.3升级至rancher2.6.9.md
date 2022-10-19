# rancher2.6.3升级至rancher2.6.9

## 备份rancher所在集群

参考：[rancher-backup使用US3备份.md](./rancher-backup使用US3备份.md)

## 下载helm chart

```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
helm pull rancher-stable/rancher
```

## 渲染模板(因为首次安装是渲染模板方式，更新也沿用)

```sh
helm template rancher ./rancher-2.6.9.tgz --output-dir . \
--namespace cattle-system \
--set hostname=rancher-apps92250.solarfs.io \
--set replicas=3 \
--set ingress.tls.source=secret \
--set useBundledSystemChart=true \
-f values.yaml
```

## 应用已渲染的模板

```
kubectl -n cattle-system apply -R -f ./rancher
```

报错

```
The Job "rancher-post-delete" is invalid: spec.template: Invalid value: core.PodTemplateSpec{ObjectMeta:v1.ObjectMeta{Name:"rancher-post-delete", GenerateName:"", Namespace:"", SelfLink:"", UID:"", ResourceVersion:"", Generation:0, CreationTimestamp:v1.Time{Time:time.Time{wall:0x0, ext:0, loc:(*time.Location)(nil)}}, DeletionTimestamp:(*v1.Time)(nil), DeletionGracePeriodSeconds:(*int64)(nil), Labels:map[string]string{"app":"rancher", "chart":"rancher-2.6.9", "controller-uid":"b6289828-e90f-41c9-b8be-ef9a639bcab8", "heritage":"Helm", "job-name":"rancher-post-delete", "release":"rancher"}, Annotations:map[string]string(nil), OwnerReferences:[]v1.OwnerReference(nil), Finalizers:[]string(nil), ClusterName:"", ManagedFields:[]v1.ManagedFieldsEntry(nil)}, Spec:core.PodSpec{Volumes:[]core.Volume{core.Volume{Name:"config-volume", VolumeSource:core.VolumeSource{HostPath:(*core.HostPathVolumeSource)(nil), EmptyDir:(*core.EmptyDirVolumeSource)(nil), GCEPersistentDisk:(*core.GCEPersistentDiskVolumeSource)(nil), AWSElasticBlockStore:(*core.AWSElasticBlockStoreVolumeSource)(nil), GitRepo:(*core.GitRepoVolumeSource)(nil), Secret:(*core.SecretVolumeSource)(nil), NFS:(*core.NFSVolumeSource)(nil), ISCSI:(*core.ISCSIVolumeSource)(nil), Glusterfs:(*core.GlusterfsVolumeSource)(nil), PersistentVolumeClaim:(*core.PersistentVolumeClaimVolumeSource)(nil), RBD:(*core.RBDVolumeSource)(nil), Quobyte:(*core.QuobyteVolumeSource)(nil), FlexVolume:(*core.FlexVolumeSource)(nil), Cinder:(*core.CinderVolumeSource)(nil), CephFS:(*core.CephFSVolumeSource)(nil), Flocker:(*core.FlockerVolumeSource)(nil), DownwardAPI:(*core.DownwardAPIVolumeSource)(nil), FC:(*core.FCVolumeSource)(nil), AzureFile:(*core.AzureFileVolumeSource)(nil), ConfigMap:(*core.ConfigMapVolumeSource)(0xc050f0e780), VsphereVolume:(*core.VsphereVirtualDiskVolumeSource)(nil), AzureDisk:(*core.AzureDiskVolumeSource)(nil), PhotonPersistentDisk:(*core.PhotonPersistentDiskVolumeSource)(nil), Projected:(*core.ProjectedVolumeSource)(nil), PortworxVolume:(*core.PortworxVolumeSource)(nil), ScaleIO:(*core.ScaleIOVolumeSource)(nil), StorageOS:(*core.StorageOSVolumeSource)(nil), CSI:(*core.CSIVolumeSource)(nil), Ephemeral:(*core.EphemeralVolumeSource)(nil)}}}, InitContainers:[]core.Container(nil), Containers:[]core.Container{core.Container{Name:"rancher-post-delete", Image:"rancher/shell:v0.1.18", Command:[]string{"/scripts/post-delete-hook.sh"}, Args:[]string(nil), WorkingDir:"", Ports:[]core.ContainerPort(nil), EnvFrom:[]core.EnvFromSource(nil), Env:[]core.EnvVar{core.EnvVar{Name:"NAMESPACES", Value:"cattle-fleet-system cattle-system rancher-operator-system", ValueFrom:(*core.EnvVarSource)(nil)}, core.EnvVar{Name:"RANCHER_NAMESPACE", Value:"cattle-system", ValueFrom:(*core.EnvVarSource)(nil)}, core.EnvVar{Name:"TIMEOUT", Value:"120", ValueFrom:(*core.EnvVarSource)(nil)}, core.EnvVar{Name:"IGNORETIMEOUTERROR", Value:"false", ValueFrom:(*core.EnvVarSource)(nil)}}, Resources:core.ResourceRequirements{Limits:core.ResourceList(nil), Requests:core.ResourceList(nil)}, VolumeMounts:[]core.VolumeMount{core.VolumeMount{Name:"config-volume", ReadOnly:false, MountPath:"/scripts", SubPath:"", MountPropagation:(*core.MountPropagationMode)(nil), SubPathExpr:""}}, VolumeDevices:[]core.VolumeDevice(nil), LivenessProbe:(*core.Probe)(nil), ReadinessProbe:(*core.Probe)(nil), StartupProbe:(*core.Probe)(nil), Lifecycle:(*core.Lifecycle)(nil), TerminationMessagePath:"/dev/termination-log", TerminationMessagePolicy:"File", ImagePullPolicy:"IfNotPresent", SecurityContext:(*core.SecurityContext)(0xc05125dec0), Stdin:false, StdinOnce:false, TTY:false}}, EphemeralContainers:[]core.EphemeralContainer(nil), RestartPolicy:"OnFailure", TerminationGracePeriodSeconds:(*int64)(0xc04e458350), ActiveDeadlineSeconds:(*int64)(nil), DNSPolicy:"ClusterFirst", NodeSelector:map[string]string(nil), ServiceAccountName:"rancher-post-delete", AutomountServiceAccountToken:(*bool)(nil), NodeName:"", SecurityContext:(*core.PodSecurityContext)(0xc050d27300), ImagePullSecrets:[]core.LocalObjectReference(nil), Hostname:"", Subdomain:"", SetHostnameAsFQDN:(*bool)(nil), Affinity:(*core.Affinity)(nil), SchedulerName:"default-scheduler", Tolerations:[]core.Toleration(nil), HostAliases:[]core.HostAlias(nil), PriorityClassName:"", Priority:(*int32)(nil), PreemptionPolicy:(*core.PreemptionPolicy)(nil), DNSConfig:(*core.PodDNSConfig)(nil), ReadinessGates:[]core.PodReadinessGate(nil), RuntimeClassName:(*string)(nil), Overhead:core.ResourceList(nil), EnableServiceLinks:(*bool)(nil), TopologySpreadConstraints:[]core.TopologySpreadConstraint(nil)}}: field is immutable
```

删除job `rancher-post-delete` 

```sh
kubectl delete job rancher-post-delete -n cattle-system
```

重新执行

```sh
kubectl -n cattle-system apply -R -f ./rancher
```

检查

```sh
kubectl get pod -n cattle-system

NAME                               READY   STATUS      RESTARTS   AGE
helm-operation-kl528               0/2     Completed   0          37m
helm-operation-mmp8h               1/2     Error       0          35m
helm-operation-r56z7               0/2     Completed   0          27m
helm-operation-rxqwp               0/2     Completed   0          36m
rancher-85874f97bb-5p46j           1/1     Running     0          41m
rancher-85874f97bb-gz56s           1/1     Running     0          41m
rancher-85874f97bb-lrx58           1/1     Running     0          37m
rancher-post-delete--1-ghvf6       0/1     Completed   0          40m
rancher-webhook-84f5b77579-rgsnq   1/1     Running     0          35m
```

其中有一个pod 错误，查看相应日志, 显示有其它进行在执行(install/upgrade/rollback)

```sh
kubectl logs helm-operation-mmp8h -n cattle-system -c helm

# 显示如下
helm upgrade --force-adopt=true --history-max=5 --install=true --namespace=cattle-fleet-system --reset-values=true --timeout=5m0s --values=/home/shell/helm/values-fleet-100.0.2-up0.3.8.yaml --version=100.0.2+up0.3.8 --wait=true fleet /home/shell/helm/fleet-100.0.2-up0.3.8.tgz
Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress
```

查看 helm-operator-kl528 pod ,日志显示如下表示正常，上面异常pod 可以忽略了

```sh
kubectl logs helm-operation-kl528 -n cattle-system -c helm

# 显示如下
...
---------------------------------------------------------------------
SUCCESS: helm upgrade --force-adopt=true --history-max=5 --install=true --namespace=cattle-fleet-system --reset-values=true --timeout=5m0s --values=/home/shell/helm/values-fleet-100.0.2-up0.3.8.yaml --version=100.0.2+up0.3.8 --wait=true fleet /home/shell/helm/fleet-100.0.2-up0.3.8.tgz
---------------------------------------------------------------------
```

## 参考

https://docs.ranchermanager.rancher.io/zh/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster/upgrades

https://docs.ranchermanager.rancher.io/zh/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster/air-gapped-upgrades

