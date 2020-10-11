
<h2><b>Webhooks using operator-sdk</b></h2>

This is a guide to creating and deploying the memcached operator sample with webhook from scratch using the operator-sdk. The base documentations used are  <a href="https://sdk.operatorframework.io/docs/building-operators/golang/quickstart/">quickstart</a> and <a href="https://sdk.operatorframework.io/docs/building-operators/golang/webhooks/">admission webhook</a> from the operator-sdk official documentation.

The complete project can be found <a href="https://github.com/sruthakeerthikotla/memcachedwebhook">here</a> .

First let's start with creating a sample memcached operator.

```
mkdir memcached-operator
cd memcached-operator
operator-sdk init --domain=example.com --repo=github.com/example-inc/memcached-operator
```

Create a Memcached API 

```
operator-sdk create api --group cache --version v1 --kind Memcached --resource=true --controller=true
```

Build and push the operator image

```
make docker-build docker-push IMG=<some-registry>/<project-name>:<tag>
```


Install the CRD and deploy the project to the cluster. Set IMG with make deploy to use the image you just pushed:

```
make install
make deploy IMG=<some-registry>/<project-name>:<tag>
```

Create a sample CR:

```
kubectl apply -f config/samples/cache_v1_memcached.yaml 
```

Verify that your operator is up and running by checking your operator pod's logs and status.
This marks the end of the basic operator setup.

You can delete the deployed operator, CR and CRD by 

```
kubectl delete -f config/samples/cache_v1_memcached.yaml
kustomize build config/default | kubectl delete -f -
```


<h4>Operator with webhook</h4>

Two types of admission webhooks can be added to the operator using the oeprator-sdk : validating and mutating. Validating webhook is used to only accept or reject the incoming CR based on some validation logic while the mutating webhook is used to amke changes to the incoming CR.

To the above operator code, add a webhook by running the following command :

```
operator-sdk create webhook --group cache --version v1 --kind Memcached --defaulting --programmatic-validation
```

Here the "--defaulting" option adds a provision for mutating webhook while "--programmatic-validation" adds code for a validation webhook.

>>Note: The webhook thus created is only applicable for CRs of kind "Memcached". CRs of other kinds cannot be handled by the webhook created in the above fashion.

The actual webhook logic can be found in the file `api/v1/memcached_webhook.go` . The contents of the file are below :


```
/*


Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package v1

import (
	"fmt"
	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/webhook"
)

// log is for logging in this package.
var memcachedlog = logf.Log.WithName("memcached-resource")

func (r *Memcached) SetupWebhookWithManager(mgr ctrl.Manager) error {
	return ctrl.NewWebhookManagedBy(mgr).
		For(r).
		Complete()
}

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!

// +kubebuilder:webhook:path=/mutate-cache-example-com-v1-memcached,mutating=true,failurePolicy=fail,groups=cache.example.com,resources=memcacheds,verbs=create;update,versions=v1,name=mmemcached.kb.io

var _ webhook.Defaulter = &Memcached{}

// Default implements webhook.Defaulter so a webhook will be registered for the type
func (r *Memcached) Default() {
	memcachedlog.Info("default", "name", r.Name)

	// TODO(user): fill in your defaulting logic.
}

// TODO(user): change verbs to "verbs=create;update;delete" if you want to enable deletion validation.
// +kubebuilder:webhook:verbs=create;update,path=/validate-cache-example-com-v1-memcached,mutating=false,failurePolicy=fail,groups=cache.example.com,resources=memcacheds,versions=v1,name=vmemcached.kb.io

var _ webhook.Validator = &Memcached{}

// ValidateCreate implements webhook.Validator so a webhook will be registered for the type
func (r *Memcached) ValidateCreate() error {
	memcachedlog.Info("validate create", "name", r.Name)
	fmt.Println("Hey there from ValidateCreate!")
	// TODO(user): fill in your validation logic upon object creation.
	return nil
}

// ValidateUpdate implements webhook.Validator so a webhook will be registered for the type
func (r *Memcached) ValidateUpdate(old runtime.Object) error {
	memcachedlog.Info("validate update", "name", r.Name)

	// TODO(user): fill in your validation logic upon object update.
	return nil
}

// ValidateDelete implements webhook.Validator so a webhook will be registered for the type
func (r *Memcached) ValidateDelete() error {
	memcachedlog.Info("validate delete", "name", r.Name)

	// TODO(user): fill in your validation logic upon object deletion.
	return nil
}
```

Let's discuss a little about the contents of this file. 

We have added a line to print a test message in the `ValidateCreate` method to check our webhook in the above code.

The webhook is managed by the same setup manager that manages the operator as well. Some attention has to be given to the "//kubebuilder" lines here.

```
// +kubebuilder:webhook:path=/mutate-cache-example-com-v1-memcached,mutating=true,failurePolicy=fail,groups=cache.example.com,resources=memcacheds,verbs=create;update,versions=v1,name=mmemcached.kb.io

// +kubebuilder:webhook:verbs=create;update,path=/validate-cache-example-com-v1-memcached,mutating=false,failurePolicy=fail,groups=cache.example.com,resources=memcacheds,versions=v1,name=vmemcached.kb.io

```

These are the properties that will be used when creating webhook configurations.

The function `Default` is where the mutating logic has to be filled in. Validation is handled in 3 different functions : `ValidateCreate`, `ValidateUpdate` and `ValidateDelete`. These functions are respectively for validation when a CR is created, updated and deleted.

>>Note: To enable `ValidateDelete` to be actually executed when a CR is deleted, you have to add "delete" to the kubebuilder verbs like "verbs=create;update;delete". By default, only create and update are enabled.


Now execute 

```
make manifests
```

This command generates the webhook manifests and enables the deployment. The webhook configurations created can be found in `config/webhook/manifests.yaml`

The webhook server accepts only those webhook configurations that provide a valid `caBundle` value. The below steps are required to setup a certificate manager, create a self signed certificate and to do a ca-injection into the webhook configurations. 

To enable cert-manager and webhook deployments, you will have to modify the file `config/default/kustomization.yaml` by uncommenting the `[WEBHOOK]` and `[CERTMANAGER]`. Your file should look like below :


```
# Adds namespace to all resources.
namespace: memcached-operator-system

# Value of this field is prepended to the
# names of all resources, e.g. a deployment named
# "wordpress" becomes "alices-wordpress".
# Note that it should also match with the prefix (text before '-') of the namespace
# field above.
namePrefix: memcached-operator-

# Labels to add to all resources and selectors.
#commonLabels:
#  someName: someValue

bases:
- ../crd
- ../rbac
- ../manager
# [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix including the one in
# crd/kustomization.yaml
- ../webhook
# [CERTMANAGER] To enable cert-manager, uncomment all sections with 'CERTMANAGER'. 'WEBHOOK' components are required.
- ../certmanager
# [PROMETHEUS] To enable prometheus monitor, uncomment all sections with 'PROMETHEUS'.
#- ../prometheus

patchesStrategicMerge:
  # Protect the /metrics endpoint by putting it behind auth.
  # If you want your controller-manager to expose the /metrics
  # endpoint w/o any authn/z, please comment the following line.
- manager_auth_proxy_patch.yaml

# [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix including the one in
# crd/kustomization.yaml
- manager_webhook_patch.yaml

# [CERTMANAGER] To enable cert-manager, uncomment all sections with 'CERTMANAGER'.
# Uncomment 'CERTMANAGER' sections in crd/kustomization.yaml to enable the CA injection in the admission webhooks.
# 'CERTMANAGER' needs to be enabled to use ca injection
- webhookcainjection_patch.yaml

# the following config is for teaching kustomize how to do var substitution
vars:
# [CERTMANAGER] To enable cert-manager, uncomment all sections with 'CERTMANAGER' prefix.
- name: CERTIFICATE_NAMESPACE # namespace of the certificate CR
  objref:
   kind: Certificate
   group: cert-manager.io
   version: v1alpha2
   name: serving-cert # this name should match the one in certificate.yaml
  fieldref:
   fieldpath: metadata.namespace
- name: CERTIFICATE_NAME
  objref:
   kind: Certificate
   group: cert-manager.io
   version: v1alpha2
   name: serving-cert # this name should match the one in certificate.yaml
- name: SERVICE_NAMESPACE # namespace of the service
  objref:
   kind: Service
   version: v1
   name: webhook-service
  fieldref:
   fieldpath: metadata.namespace
- name: SERVICE_NAME
  objref:
   kind: Service
   version: v1
   name: webhook-service
```

In addition the `[CERTMANAGER]` section in `config/crd/kustomization.yaml` has to uncommented. Your file would then look like this :

```
# This kustomization.yaml is not intended to be run by itself,
# since it depends on service name and namespace that are out of this kustomize package.
# It should be run by config/default
resources:
- bases/cache.example.com_memcacheds.yaml
# +kubebuilder:scaffold:crdkustomizeresource

patchesStrategicMerge:
# [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix.
# patches here are for enabling the conversion webhook for each CRD
#- patches/webhook_in_memcacheds.yaml
# +kubebuilder:scaffold:crdkustomizewebhookpatch

# [CERTMANAGER] To enable webhook, uncomment all the sections with [CERTMANAGER] prefix.
# patches here are for enabling the CA injection for each CRD
- patches/cainjection_in_memcacheds.yaml
# +kubebuilder:scaffold:crdkustomizecainjectionpatch

# the following config is for teaching kustomize how to do kustomization for CRDs.
configurations:
- kustomizeconfig.yaml

```


>The `[WEBHOOK]` section in the above file must be uncommented if you want to include a <b>conversion webhook</b>. We will not be doing it in this guide.


To setup the certificate manager in your cluster, execute :

```
oc apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.1/cert-manager.yaml
```

The certficate and issuer details can be found in `config/certmanager/certificate.yaml`. 

Now build your operator image, push the to a repository and deploy it to your cluster using the same commands like for the sample operator above, i.e. 

```
make docker-build docker-push IMG=<some-registry>/<project-name>:<tag>
make deploy IMG=<some-registry>/<project-name>:<tag>

```

Your operator deployment logs should now show the webhook server starting as well like below :

```
I1008 05:06:55.803629       1 request.go:621] Throttling request took 1.002622548s, request: GET:https://172.30.0.1:443/apis/cert-manager.io/v1?timeout=32s
2020-10-08T05:06:58.313Z	INFO	controller-runtime.metrics	metrics server is starting to listen	{"addr": "127.0.0.1:8080"}
2020-10-08T05:06:58.313Z	INFO	controller-runtime.builder	Registering a mutating webhook	{"GVK": "cache.example.com/v1, Kind=Memcached", "path": "/mutate-cache-example-com-v1-memcached"}
2020-10-08T05:06:58.313Z	INFO	controller-runtime.webhook	registering webhook	{"path": "/mutate-cache-example-com-v1-memcached"}
2020-10-08T05:06:58.313Z	INFO	controller-runtime.builder	Registering a validating webhook	{"GVK": "cache.example.com/v1, Kind=Memcached", "path": "/validate-cache-example-com-v1-memcached"}
2020-10-08T05:06:58.314Z	INFO	controller-runtime.webhook	registering webhook	{"path": "/validate-cache-example-com-v1-memcached"}
2020-10-08T05:06:58.314Z	INFO	setup	starting manager
I1008 05:06:58.314263       1 leaderelection.go:242] attempting to acquire leader lease  memcached-operator-system/f1c5ece8.example.com...
2020-10-08T05:06:58.319Z	INFO	controller-runtime.manager	starting metrics server	{"path": "/metrics"}
2020-10-08T05:06:58.319Z	INFO	controller-runtime.webhook.webhooks	starting webhook server
I1008 05:06:58.399489       1 leaderelection.go:252] successfully acquired lease memcached-operator-system/f1c5ece8.example.com
2020-10-08T05:06:58.399Z	DEBUG	controller-runtime.manager.events	Normal	{"object": {"kind":"ConfigMap","namespace":"memcached-operator-system","name":"f1c5ece8.example.com","uid":"7a842866-9646-41a8-89c0-da91d7ee2371","apiVersion":"v1","resourceVersion":"20958774"}, "reason": "LeaderElection", "message": "memcached-operator-controller-manager-7c6c84c59d-wjlpd_429c51cb-47cf-4ced-898a-008e0fa8056b became leader"}
2020-10-08T05:06:58.399Z	INFO	controller-runtime.certwatcher	Updated current TLS certificate
2020-10-08T05:06:58.399Z	INFO	controller	Starting EventSource	{"reconcilerGroup": "cache.example.com", "reconcilerKind": "Memcached", "controller": "memcached", "source": "kind source: /, Kind="}
2020-10-08T05:06:58.399Z	INFO	controller-runtime.webhook	serving webhook server	{"host": "", "port": 9443}
2020-10-08T05:06:58.400Z	INFO	controller-runtime.certwatcher	Starting certificate watcher
```

To verify that your webhook configuration is being applied, apply a memcached CR :

```
oc apply -f config/samples/cache_v1_memcached.yaml
```

On your pod you should be able to see the log added `"Hey there from ValidateCreate!"`

