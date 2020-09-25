# kubevirt api server 证书过期问题导致openshfit调度异常

## openshfit jenkins ci 创建build时报错

```
Starting the "Trigger OpenShift Build" step with build config "rnode-pkg-build" from the project "pld-cicd".

An exception occurred invoking a REST operation against the OpenShift master.  The operation will be retried.  Exception message "Exception trying to GET https://172.30.0.1/apis/subresources.kubevirt.io/v1alpha3 response code: 503".

An exception occurred invoking a REST operation against the OpenShift master.  The operation will be retried.  Exception message "Exception trying to GET https://172.30.0.1/apis/subresources.kubevirt.io/v1alpha3 response code: 503".

An exception occurred invoking a REST operation against the OpenShift master.  The operation will be retried.  Exception message "Exception trying to GET https://172.30.0.1/apis/subresources.kubevirt.io/v1alpha3 response code: 503".

After a few retries, giving up invoking the REST operation against the OpenShift master.  Final exception message "Exception trying to GET https://172.30.0.1/apis/subresources.kubevirt.io/v1alpha3 response code: 503".
com.openshift.restclient.OpenShiftException: Exception trying to GET https://172.30.0.1/apis/subresources.kubevirt.io/v1alpha3 response code: 503
	at com.openshift.internal.restclient.okhttp.ResponseCodeInterceptor.createOpenShiftException(ResponseCodeInterceptor.java:114)
	at com.openshift.internal.restclient.okhttp.ResponseCodeInterceptor.intercept(ResponseCodeInterceptor.java:65)
	at okhttp3.RealCall$ApplicationInterceptorChain.proceed(RealCall.java:190)
	at okhttp3.RealCall.getResponseWithInterceptorChain(RealCall.java:163)
	at okhttp3.RealCall.execute(RealCall.java:57)
	at com.openshift.internal.restclient.ApiTypeMapper.readEndpoint(ApiTypeMapper.java:193)
	at com.openshift.internal.restclient.ApiTypeMapper.getResources(ApiTypeMapper.java:168)
	at com.openshift.internal.restclient.ApiTypeMapper.lambda$null$2(ApiTypeMapper.java:127)
	at java.util.ArrayList.forEach(ArrayList.java:1257)
	at com.openshift.internal.restclient.ApiTypeMapper.lambda$init$3(ApiTypeMapper.java:126)
	at java.util.ArrayList.forEach(ArrayList.java:1257)
	at com.openshift.internal.restclient.ApiTypeMapper.init(ApiTypeMapper.java:124)
	at com.openshift.internal.restclient.ApiTypeMapper.isSupported(ApiTypeMapper.java:111)
	at com.openshift.internal.restclient.URLBuilder.buildWithNamespaceInPath(URLBuilder.java:145)
	at com.openshift.internal.restclient.URLBuilder.build(URLBuilder.java:132)
	at com.openshift.internal.restclient.DefaultClient.execute(DefaultClient.java:251)
	at com.openshift.internal.restclient.DefaultClient.execute(DefaultClient.java:222)
	at com.openshift.internal.restclient.DefaultClient.execute(DefaultClient.java:210)
	at com.openshift.internal.restclient.DefaultClient.get(DefaultClient.java:333)
	at com.openshift.jenkins.plugins.pipeline.model.RetryIClient.lambda$get$5(RetryIClient.java:189)
	at com.openshift.jenkins.plugins.pipeline.model.RetryIClient.retry(RetryIClient.java:78)
	at com.openshift.jenkins.plugins.pipeline.model.RetryIClient.get(RetryIClient.java:189)
	at com.openshift.jenkins.plugins.pipeline.model.IOpenShiftBuilder.coreLogic(IOpenShiftBuilder.java:273)
	at com.openshift.jenkins.plugins.pipeline.model.IOpenShiftPlugin.doItCore(IOpenShiftPlugin.java:359)
	at com.openshift.jenkins.plugins.pipeline.dsl.OpenShiftBuilderExecution.run(OpenShiftBuilderExecution.java:45)
	at com.openshift.jenkins.plugins.pipeline.dsl.OpenShiftBuilderExecution.run(OpenShiftBuilderExecution.java:17)
	at org.jenkinsci.plugins.workflow.steps.AbstractSynchronousNonBlockingStepExecution$1$1.call(AbstractSynchronousNonBlockingStepExecution.java:47)
	at hudson.security.ACL.impersonate(ACL.java:290)
	at org.jenkinsci.plugins.workflow.steps.AbstractSynchronousNonBlockingStepExecution$1.run(AbstractSynchronousNonBlockingStepExecution.java:44)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
```

##  在master手动访问报错

```
curl -k https://172.30.0.1/apis/subresources.kubevirt.io/v1alpha3
```

```
Error: 'x509: certificate has expired or is not yet valid'
Trying to reach: 'https://172.30.196.33:443/apis/subresources.kubevirt.io/v1alpha3'
```

## 清理kubevirt.io 相关secrets

```
kubectl delete secrets --namespace kubevirt -l kubevirt.io
```

输出删除的secrets列表

```
secret "kubevirt-operator-certs" deleted
secret "kubevirt-virt-api-certs" deleted
secret "kubevirt-virt-handler-certs" deleted
```

## 删除kubevirt.io 相关pod, 重建pod和secret

```
kubectl delete pods --namespace kubevirt -l kubevirt.io
```
输出删除的pod列表

```
pod "virt-api-6db54c64b5-c27sd" deleted
pod "virt-api-6db54c64b5-mngnl" deleted
pod "virt-controller-7cfdbfc5cd-654jx" deleted
pod "virt-controller-7cfdbfc5cd-9mc7g" deleted
pod "virt-controller-7cfdbfc5cd-l7gzw" deleted
pod "virt-handler-gqlws" deleted
pod "virt-handler-qx6wg" deleted
pod "virt-operator-5d65797b8-n6l87" deleted
pod "virt-operator-5d65797b8-xc2qn" deleted
```

## 手动访问api测试

```
curl -k https://172.30.0.1/apis/subresources.kubevirt.io/v1alpha3
```

输出正确内容

```
{
 "kind": "APIResourceList",
 "apiVersion": "v1alpha3",
 "groupVersion": "subresources.kubevirt.io/v1alpha3",
 "resources": [
  {
   "name": "virtualmachineinstances/vnc",
   "singularName": "",
   "namespaced": true,
   "kind": "",
   ...
```

## 重建构建jenkins任务正常

## 参考

https://github.com/kubevirt/kubevirt/issues/3369#issuecomment-666306190