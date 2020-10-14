<h2><b>Deploying operator webhooks with OLM</b></h2>

The Operator Lifecycle Manager (OLM) extends Kubernetes to provide a declarative way to install, manage, and upgrade Operators on a cluster. Using OLM, we can create a catalog source that holds multiple operators that can be made available for installation from the Operator Hub. We will be using this approach (of many) of OLM. In depth understanding on OLM can be found <a href="https://olm.operatorframework.io/">here</a>

To install OLM in your cluster, execute

```
export olm_release=0.15.1
kubectl apply -f https://github.com/operator-framework/operator-lifecycle-manager/releases/download/${olm_release}/crds.yaml
kubectl apply -f https://github.com/operator-framework/operator-lifecycle-manager/releases/download/${olm_release}/olm.yaml
```

More detailed installation for OLM are available <a href="https://olm.operatorframework.io/docs/getting-started/">here</a>

We will continue using the sample memcached operator created in this <a href="https://medium.com/@sruthakeerthi.k/webhook-with-operator-sdk-4476bf1522ed">blog</a>. The complete github project is available <a href="https://github.com/sruthakeerthikotla/memcachedwebhook">here</a>

First we have to add the apiserver certificate details into our operator. This is because OLM provides a single CA for all it's deployments which was used by APIService logic. This constraint is mentioned in detail in <a href="https://docs.openshift.com/container-platform/4.5/operators/user/olm-webhooks.html#olm-webhook-considerations_olm-webhooks">this</a> document. 
Add the below code in the file `"main.go"`

```
	const (
		WebhookPort     = 4343 //You can specify any port value here
		WebhookCertDir  = "/apiserver.local.config/certificates"
		WebhookCertName = "apiserver.crt"
		WebhookKeyName  = "apiserver.key"
	)

	srv := mgr.GetWebhookServer()
	srv.CertDir = WebhookCertDir
	srv.CertName = WebhookCertName
	srv.KeyName = WebhookKeyName
	srv.Port = WebhookPort
```

Build and deploy the operator with webhook.

```
make docker-build docker-push IMG=<some-registry>/<project-name>:<tag>
```

Generate the resources needed for the bundle image. The below command creates the ClusterServiceVersion(CSV) and other required files in the `bundle` folder.

```
make bundle
```

After executing the above line, there are a few modifications needed in the file `bundle/manifests/memcached-operator.clusterserviceversion.yaml`. 
  
  1. Change the values of fields `"admissionReviewVersions"` in 2 places (one for the mutating webhook configuration and one for the validating webhook configuration) from null to an array of strings that specify the versions for which the webhook needs to watch for. As an example we can change the fields as `admissionReviewVersions: ["v1"]`. This means we will be monitoring for CRs of version `"v1"`.
  2. Change the value of field `"sideEffects"` in 2 places  (one for the mutating webhook configuration and one for the validating webhook configuration) to `None` from `'null'`.
  3. Change the `"deploymentName"` of the webhook configurations(both validating and mutating) to the same name as the operator deployment
  4. Add target and container port fields. These values must have values same as the `"WebhookPort"` value set in `"main.go"` above, i.e.

  ```
	containerPort: 4343
    targetPort: 4343
  ```
  at the end of each of the webhook configuration after `webhookPath` field
  5. Modify the `"image"` field of the operator manager from `controller:latest` to your actual operator image that you have built and pushed above.
  6. Delete the volume and volumemounts sections that have been added to the CSV. They are not needed as OLM manages the certificates.

Now that the CSV has no issues, create a bundle image and push it by executing 

```
make bundle-build BUNDLE_IMG=<some-registry>/<project-name>:<bundle-tag>
docker push <some-registry>/<project-name>:<bundle-tag>
```
To bundle various bundles into a single catalog source, the <b>`opm`</b> utility can be used. <b>`opm`</b> basically generates and updates registry databases as well as the index images that encapsulate them.
opm can be installed from <a href="https://github.com/operator-framework/operator-registry/releases">releases</a> here

To create a catalog source with the bundle image created and pushed above, and to push the created catalog source to registry :

```
 opm index add --bundles <some-registry>/<project-name>:<bundle-tag> --container-tool docker -t <some-registry>/<project-name>:<catalog-tag>
 docker push <some-registry>/<project-name>:<catalog-tag>
```
>>Note: More on building an index using opm can be read <a href="https://github.com/operator-framework/operator-registry#building-an-index-of-operators-using-opm">here</a>

Now apply the below Catalog Source to your cluster by changing the required image field to the created catalog source image 

```
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: memcached-ops
  namespace: marketplace #"openshift-marketplace" in case of OCP
spec:
  displayName: Memcached Operators Catalog Source
  publisher: Mem
  sourceType: grpc
  image: <some-registry>/<project-name>:<catalog-tag>
  updateStrategy:
    registryPoll:
      interval: 45m
```

In your operator hub, search for the Memcached operator and install it. You should now see the operator installed successffully and a deployment and pod of the operator with webhook running.

>>Note: You might see an error saying "no kind admissionreview is registered for version admission.k8s.io/v1" when a Memcached CR is applied. This would be because we are not returning a fully formed AdmissionReview object from the mutating webhook logic. To avoid this, implement the mutating logic.



<h3>Some learnings</h3>

Folks searching around may find this useful looking article https://www.velotio.com/engineering-blog/managing-tls-certificate-for-kubernetes-admission-webhook - while this looks promising, for the OLM scenario this isn't ideal. 


We experimented by creating an init-container based solution that would directly create a ca cert, write this to a location, and then use the emptyDir spec on the volume mount so that other containers could read from this. We then modified the main.go container to use the Go client to modify the Mutating and Validating admission webhook configurations, to set the caBundle field to be the one we had created. 


While this "works", this does result in your operator being completely reinstalled every few minutes. 


We believe this to be due to the reconciler noticing that the pod itself has changed. 


So we advise sticking with the above recommendation: specifying the configuration in Go code, and then modifying the webhook server to use said configuration.
