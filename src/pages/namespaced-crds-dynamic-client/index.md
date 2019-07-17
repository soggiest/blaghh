---
title: Using Client-Go's dynamic client to operate namespaced CRDs 
date: "2019-07-17"
---

# Using Client-Go's dynamic client to operate namespaced CRDs

Custom Resource Definitions are a useful mechanism in Kubernetes that extends the K8S API to include new resource types. This helps a lot when writing your own controllers for Kubernetes and need to persist data outside of the controller itself. Traditionally, creating your own CRDs meant that you must implement a new resource interface similar to this:

```
type ElasticsearchClustersGetter interface {
	ElasticsearchClusters(namespace string) ElasticsearchClusterInterface
}

// ElasticsearchClusterInterface has methods to work with ElasticsearchCluster resources.
type ElasticsearchClusterInterface interface {
	Create(*v1.ElasticsearchCluster) (*v1.ElasticsearchCluster, error)
	Update(*v1.ElasticsearchCluster) (*v1.ElasticsearchCluster, error)
	UpdateStatus(*v1.ElasticsearchCluster) (*v1.ElasticsearchCluster, error)
	Delete(name string, options *meta_v1.DeleteOptions) error
	DeleteCollection(options *meta_v1.DeleteOptions, listOptions meta_v1.ListOptions) error
	Get(name string, options meta_v1.GetOptions) (*v1.ElasticsearchCluster, error)
	List(opts meta_v1.ListOptions) (*v1.ElasticsearchClusterList, error)
	Watch(opts meta_v1.ListOptions) (watch.Interface, error)
	Patch(name string, pt types.PatchType, data []byte, subresources ...string) (oof *v1.ElasticsearchCluster, err error)
	ElasticsearchClusterExpansion
}
```

This piece of code was taken from the [ElasticSearch Operator](https://github.com/upmc-enterprises/elasticsearch-operator/), and yes, I asked first before using it.

Creating an interface for your CRD is the proper discipline. However, what if you're writing a piece of code and you don't need all of the CRUD (create, read, update, delete) functions for your CRD, or you're feeling lazy and don't want to wire up a whole interferace and associated methods? 

I mean, you could just write an interface that only has the methods you actually need, but, WHATEVER.

Luckily for you there is a handy `dynamic` client for Kubernetes client-go. The `dynamic` client allows you to specify the resource you want to interact with via a GroupVersionResource schema (https://github.com/kubernetes/apimachinery/blob/master/pkg/runtime/schema), and provides a full interface for interacting with that object. This is extremely handy for interacting with CRDs, particularly if the code you're writing doesn't warrant the extra weight of an entire CRD interface.

So, let's see an example:
```
package main

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/rest"
	"k8s.io/klog"
)

var (
	oofsGVR = schema.GroupVersionResource{
		Group:    "foo-bar.com",
		Version:  "v1",
		Resource: "oof",
	}
)

func main() {
	klog.InitFlags(nil)

	klog.Infof("Starting test")
	config, _ := rest.InClusterConfig()

	dynClient, errClient := dynamic.NewForConfig(config)
	if errClient != nil {
		klog.Fatalf("Error received creating client %v", errClient)
	}

	crdClient := dynClient.Resource(oofsGVR)

	crd, errCrd := crdClient.Get("bigOof", metav1.GetOptions{})
	if errCrd != nil {
		klog.Fatalf("Error getting CRD %v", errCrd)
	}
	klog.Infof("Got CRD: %v", crd)
}

```

From this example we see a fairly simple mechanism for retrieving an `oof` Kind resource of the type `foo-bar.com` named `bigoof`. This works well if you're looking for a CRD that's scoped at the `Cluster` leverl. But, what if your CRD is scoped at the `Namespace`? Well, then you'll receive the following error:

```
F0717 01:12:15.047145       8 main.go:96] Error getting CRD Namespace parameter required.
```

Okay, so...how do you go about setting the namespace parameter? Is it in the `GroupVersionResource` schema? No, that schema only has fields for `Group`, `Version`, and `Resource`, which, honestly, you should probably have gleaned from its name. 

Well, luckily the `dynamic` client's `Resource` interface has an extra interface that normal CRD interfaces don't:

```
type NamespaceableResourceInterface interface {
	Namespace(string) ResourceInterface
	ResourceInterface
}
```

From this we can deduce that we can either add in a namespace right before the normal interface methods or just use the methods. Using this our code will look like:

```
package main

import (
        metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
        "k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
        "k8s.io/apimachinery/pkg/runtime/schema"
        "k8s.io/client-go/dynamic"
        "k8s.io/client-go/rest"
        "k8s.io/klog"
)

var (
        oofsGVR = schema.GroupVersionResource{
                Group:    "foo-bar.com",
                Version:  "v1",
                Resource: "oof",
        }
)

func main() {
        klog.InitFlags(nil)

        klog.Infof("Starting test")
        config, _ := rest.InClusterConfig()

        dynClient, errClient := dynamic.NewForConfig(config)
        if errClient != nil {
                klog.Fatalf("Error received creating client %v", errClient)
        }

        crdClient := dynClient.Resource(oofsGVR)

        crd, errCrd := crdClient.Namespace("deep13").Get("bigoof", metav1.GetOptions{})
        if errCrd != nil {
                klog.Fatalf("Error getting CRD %v", errCrd)
        }
        klog.Infof("Got CRD: %v", crd)
}
```

And now our code will return:
```
I0717 16:06:32.243481       8 main.go:103] Got CRD: &{map[apiVersion:foo-bar.com/v1 kind:Oof metadata:map[annotations:map[kubectl.kubernetes.io/last-applied-configuration:{"apiVersion":"foo-bar.com/v1","kind":"oof","metadata":{"annotations":{},"name":"bigoof","namespace":"deep13"},"spec":{"mode":"soggy"}}
] creationTimestamp:2019-07-16T23:19:37Z generation:1 name:bigoof namespace:deep13 resourceVersion:8596656 selfLink:/apis/foo-bar.com/v1/namespaces/deep13/off/bigoof uid:2dce0c5f-a820-11e9-b686-027abe3670ee] spec:map[mode:soggy]]}
```

We see that the `dynamic` client is pretty handy for doing simple read operations of our CRD, but there's a trade off if you want to do anything more complicated. Since the `dynamic` client only works with `unstructured` typed data there aren't any handy structs or methods we can leverage to easily modify our CRD. If you were to write you own CRD interface you could easily add these structs and methods, but we're not doing that here. So, what does it look like if we want to update our CRD?

```
package main

import (
        metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
        "k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
        "k8s.io/apimachinery/pkg/runtime/schema"
        "k8s.io/client-go/dynamic"
        "k8s.io/client-go/rest"
        "k8s.io/klog"
)

var (
        oofsGVR = schema.GroupVersionResource{
                Group:    "foo-bar.com",
                Version:  "v1",
                Resource: "oof",
        }
)

func main() {
        klog.InitFlags(nil)

        klog.Infof("Starting test")
        config, _ := rest.InClusterConfig()

        dynClient, errClient := dynamic.NewForConfig(config)
        if errClient != nil {
                klog.Fatalf("Error received creating client %v", errClient)
        }

        crdClient := dynClient.Resource(oofsGVR)

        crd, errCrd := crdClient.Namespace("deep13").Get("bigoof", metav1.GetOptions{})
        if errCrd != nil {
                klog.Fatalf("Error getting CRD %v", errCrd)
        }
        klog.Infof("Got CRD: %v", crd)

	resourceVer := crd.GetResourceVersion()

	updateObj := NewOofs("bigoof", "crunchy", resourceVer)
	crdUpdate, errUpdate := crdClient.Namespace("deep13").Update(updateObj, metav1.UpdateOptions{})
	if errUpdate != nil {
		klog.Fatalf("Error updating CRD", errUpdate)
	}

	klog.Infof("Updated CRD %v", crdUpdate)
}

func NewOofs(name string, mode string, resourceVer string) *unstructured.Unstructured {
	return &unstructured.Unstructured{
		Object: map[string]interface{}{
			"kind":       "Oof",
			"apiVersion": oofsGVR.Group + "/v1",
			"metadata": map[string]interface{}{
				"name":            name,
				"resourceVersion": resourceVer,
			},
			"spec": map[string]interface{}{
				"mode": mode,
			},
		},
	}
}
```

As we can see, in order to change the `mode` value for our CRD from `soggy` to `crunchy` we had to construct an entirely new `unstructured` data type and include the new value and the current CRD's resource version. Luckily, the `dynamic` client comes with some great methods to gather any necessary data from `unstructed` type data, otherwise getting the resource version would've been kind of annoying. 

The output of out new code looks like:
```
I0717 18:23:46.116109       8 main.go:103] Got CRD: &{map[apiVersion:foo-bar.com/v1 kind:Oof metadata:map[creationTimestamp:2019-07-16T23:19:37Z generation:1 name:oof namespace:deep13 resourceVersion:8787086 selfLink:/apis/foo-bar.com/v1/namespaces/deep13/results/result uid:2dce0c5f-a820-11e9-b686-027abe3670ee] spec:map[mode:soggy]]}
I0717 18:23:46.120128       8 main.go:113] Updated CRD &{map[apiVersion:foo-bar.com/v1 kind:Oof metadata:map[creationTimestamp:2019-07-16T23:19:37Z generation:1 name:oof namespace:deep13 resourceVersion:8787086 selfLink:/apis/foo-bar.com/v1/namespaces/deep13/results/result uid:2dce0c5f-a820-11e9-b686-027abe3670ee] spec:map[mode:crunchy]]}
```

As you can see the `dynamic` client is a really useful tool for manipulating CRDs. Despite what I was saying earlier you could use the dynamic client for your controller instead of writing your own interface, but know there are some trade-offs there. You've got to make the choice for which is right for you and your code, but it's awesome that we've got some choice.
