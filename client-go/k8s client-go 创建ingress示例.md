# k8s client-go 创建ingress示例

## 代码示例

```go
package main

import (
	"context"
	"flag"
	"fmt"
	networkingv1 "k8s.io/api/networking/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"os"
	"path/filepath"
)

func main() {

	var kubeconfig *string
	if home := homedir.HomeDir(); home != "" {
		kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "uk8stest.yaml"), "(optional) absolute path to the kubeconfig file")
	} else {
		kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
	}
	flag.Parse()

	// use the current context in kubeconfig
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err.Error())
	}

	//// creates the in-cluster config , 如果是集群内访问使用此函数创建 config
	//config, err := rest.InClusterConfig()
	//if err != nil {
	//	panic(err.Error())
	//}
	// creates the clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}

	// 创建Ingress对象
	ingressClassName := "nginx"
	pathTypeImplementationSpecific := networkingv1.PathTypeImplementationSpecific
	ingress := &networkingv1.Ingress{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "my-ingress",
			Namespace: "default",
		},
		Spec: networkingv1.IngressSpec{
			IngressClassName: &ingressClassName,
			Rules: []networkingv1.IngressRule{
				{
					Host: "example.com",
					IngressRuleValue: networkingv1.IngressRuleValue{
						HTTP: &networkingv1.HTTPIngressRuleValue{
							Paths: []networkingv1.HTTPIngressPath{
								{
									Path: "/",
									Backend: networkingv1.IngressBackend{
										Service: &networkingv1.IngressServiceBackend{
											Name: "my-service",
											Port: networkingv1.ServiceBackendPort{
												Number: 80,
											},
										},
									},
									PathType: &pathTypeImplementationSpecific,
								},
							},
						},
					},
				},
			},
			TLS: []networkingv1.IngressTLS{
				{
					Hosts:      []string{"example.com"},
					SecretName: "my-tls-secret",
				},
			},
		},
	}

	// 创建Ingress对象
	result, err := clientset.NetworkingV1().Ingresses("default").Create(context.Background(), ingress, metav1.CreateOptions{})
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to create Ingress: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("Created Ingress %q.\n", result.GetObjectMeta().GetName())
}

```

