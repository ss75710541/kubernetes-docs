# 解决openshift3.11不能下载redhat registry.access.redhat.com中镜像问题


尝试`docker pull`

```
# docker pull registry.access.redhat.com/openshift3/registry-console:v3.11
Trying to pull repository registry.access.redhat.com/openshift3/registry-console ...
open /etc/docker/certs.d/registry.access.redhat.com/redhat-ca.crt: no such file or directory
```

下载redhat证书

```
# wget http://mirror.centos.org/centos/7/os/x86_64/Packages/python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm
# rpm2cpio python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm | cpio -iv --to-stdout ./etc/rhsm/ca/redhat-uep.pem | tee /etc/rhsm/ca/redhat-uep.pem
```

重试后显示正常

```
# docker pull registry.access.redhat.com/openshift3/registry-console:v3.11
Trying to pull repository registry.access.redhat.com/openshift3/registry-console ...
v3.11: Pulling from registry.access.redhat.com/openshift3/registry-console
Digest: sha256:571773b087790ed2ac21a47c5abd89308c54ec53a7f9a1d17b664b135191862e
Status: Image is up to date for registry.access.redhat.com/openshift3/registry-console:v3.11
```