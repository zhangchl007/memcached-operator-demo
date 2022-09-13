# memcached-operator-demo
```
go version 1.17
operator-sdk v1.19.1
```
# The training video

[operator Demo](https://www.bilibili.com/video/BV1xg411U7xa/?vd_source=2a8b5d42d59a0cb370122ff08a2e002f)

[operator olm Demo](https://www.bilibili.com/video/BV1Te4y1a7HQ/?vd_source=2a8b5d42d59a0cb370122ff08a2e002f)

# operator ops 

```
mkdir memcached-operator
cd memcached-operator
operator-sdk init --domain example.com --repo github.com/example/memcached-operator

operator-sdk create api --group cache --version v1alpha1 --kind Memcached --resource --controller

# modify api and controller

type MemcachedSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Foo is an example field of Memcached. Edit memcached_types.go to remove/update
	Size int32 `json:"size"`
}

// MemcachedStatus defines the observed state of Memcached
type MemcachedStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file
	Nodes []string `json:"nodes"`
}

```

# generate manifest

make generate
make manifests
make undeploy

```
# make operator docker image

make docker-build IMG=quay.io/zhangchl007/memcached-operator:v0.0.3

make deploy IMG=quay.io/zhangchl007/memcached-operator:v0.0.3

# if the error show operator sa can't list deployments

cat config/rbac/memcached_role.yaml 
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  creationTimestamp: null
  name: memcached-operator-manager-role
rules:
- apiGroups:
  - cache.example.com
  - apps
  resources:
  - memcacheds
  - deployments
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
- apiGroups:
  - cache.example.com
  - apps
  resources:
  - memcacheds/finalizers
  - deployments
  verbs:
  - update
- apiGroups:
  - cache.example.com
  - apps
  resources:
  - memcacheds/status
  - deployments
  verbs:
  - get
  - patch
  - update

oc apply -f config/rbac/memcached_role.yaml

```
# deploy memcached cluster

kc apply -f config/samples/cache_v1alpha1_memcached.yaml

# deploy cert-manager

```
wget  https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.yaml
kc apply -f cert-manager.yaml
kc -n cert-manager get pods
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-6544c44c6b-996jm              1/1     Running   0          96m
cert-manager-cainjector-5687864d5f-fwkpg   1/1     Running   0          96m
cert-manager-webhook-785bb86798-j599k      1/1     Running   0          96m

```
# enalbe validating admission webhook

```
operator-sdk create webhook --group cache --version v1alpha1 --kind Memcached --defaulting --programmatic-validation

cat config/default/kustomization.yaml 
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
- ../prometheus

patchesStrategicMerge:
# Protect the /metrics endpoint by putting it behind auth.
# If you want your controller-manager to expose the /metrics
# endpoint w/o any authn/z, please comment the following line.
- manager_auth_proxy_patch.yaml

# Mount the controller config file for loading manager configurations
# through a ComponentConfig type
#- manager_config_patch.yaml

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
- name: CERTIFICATE_NAMESPACE  #namespace of the certificate CR
  objref:
    kind: Certificate
    group: cert-manager.io
    version: v1
    name: serving-cert  #this name should match the one in certificate.yaml
  fieldref:
    fieldpath: metadata.namespace
- name: CERTIFICATE_NAME
  objref:
    kind: Certificate
    group: cert-manager.io
    version: v1
    name: serving-cert  #this name should match the one in certificate.yaml
- name: SERVICE_NAMESPACE  #namespace of the service
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

# generate manifest

make manifests
make docker-build IMG=quay.io/zhangchl007/memcached-operator:v0.0.4
make deploy  IMG=quay.io/zhangchl007/memcached-operator:v0.0.4

```
# how validating and mutating admission webhooks work

```
just put a little coding below:

//+kubebuilder:webhook:path=/mutate-cache-example-com-v1alpha1-memcached,mutating=true,failurePolicy=fail,sideEffects=None,groups=cache.example.com,resources=memcacheds,verbs=create;update,versions=v1alpha1,name=mmemcached.kb.io,admissionReviewVersions=v1

var _ webhook.Defaulter = &Memcached{}

// Default implements webhook.Defaulter so a webhook will be registered for the type
func (r *Memcached) Default() {
	memcachedlog.Info("default", "name", r.Name)

	// TODO(user): fill in your defaulting logic.
	if r.Spec.Size < 3 {
		r.Spec.Size = 1
		memcachedlog.Info("Memcached.Spec.Size has been modified by mutating admission webhook %s", "Size", r.Spec.Size)
	}
}

// TODO(user): change verbs to "verbs=create;update;delete" if you want to enable deletion validation.
//+kubebuilder:webhook:path=/validate-cache-example-com-v1alpha1-memcached,mutating=false,failurePolicy=fail,sideEffects=None,groups=cache.example.com,resources=memcacheds,verbs=create;update,versions=v1alpha1,name=vmemcached.kb.io,admissionReviewVersions=v1

var _ webhook.Validator = &Memcached{}

// ValidateCreate implements webhook.Validator so a webhook will be registered for the type
func (r *Memcached) ValidateCreate() error {
	memcachedlog.Info("validate create", "name", r.Name)

	// TODO(user): fill in your validation logic upon object creation.
	key := "inject"
	anno, found := r.Annotations[key]
	if !found {
		memcachedlog.Info("missing annotation %s", "key", key)
	}
	if len(anno) == 0 {
		memcachedlog.Info("missing annotation %s", "key", anno)

	}

	return nil
}

```
# webhook logging 

```

1.6618807272097151e+09	DEBUG	controller-runtime.webhook.webhooks	received request	{"webhook": "/mutate-cache-example-com-v1alpha1-memcached", "UID": "0419015e-74b8-46ef-bfa6-abfe001c95cf", "kind": "cache.example.com/v1alpha1, Kind=Memcached", "resource": {"group":"cache.example.com","version":"v1alpha1","resource":"memcacheds"}}
1.6618807272098625e+09	INFO	memcached-resource	default	{"name": "memcached-sample"}
1.6618807272098734e+09	INFO	memcached-resource	Memcached.Spec.Size has been modified by mutating admission webhook %s	{"Size": 1}
1.6618807272100334e+09	DEBUG	controller-runtime.webhook.webhooks	wrote response	{"webhook": "/mutate-cache-example-com-v1alpha1-memcached", "code": 200, "reason": "", "UID": "0419015e-74b8-46ef-bfa6-abfe001c95cf", "allowed": true}
1.661880727216337e+09	DEBUG	controller-runtime.webhook.webhooks	received request	{"webhook": "/validate-cache-example-com-v1alpha1-memcached", "UID": "690af189-3212-4527-b494-50816f478fa0", "kind": "cache.example.com/v1alpha1, Kind=Memcached", "resource": {"group":"cache.example.com","version":"v1alpha1","resource":"memcacheds"}}
1.6618807272164948e+09	INFO	memcached-resource	validate create	{"name": "memcached-sample"}
1.6618807272165124e+09	INFO	memcached-resource	missing annotation %s	{"key": "inject"}
1.6618807272165198e+09	INFO	memcached-resource	missing annotation %s	{"key": ""}
1.661880727216553e+09	DEBUG	controller-runtime.webhook.webhooks	wrote response	{"webhook": "/validate-cache-example-com-v1alpha1-memcached", "code": 200, "reason": "", "UID": "690af189-3212-4527-b494-50816f478fa0", "allowed": true}
1.6618807272227423e+09	INFO	controller.memcached	Creating a new Deployment	{"reconciler group": "cache.example.com", "reconciler kind": "Memcached", "name": "memcached-sample", "namespace": "default", "Deployment.Namespace": "default", "Deployment.Name": "memcached-sample"

```
# Operator and OLM integration
```
kc apply -f crds.yaml
kc apply -f olm.yaml
operator-sdk olm status

export USERNAME=zhangchl007
export VERSION=0.0.7
export IMG=quay.io/$USERNAME/memcached-operator:v$VERSION # location where your operator image is hosted
export BUNDLE_IMG=quay.io/$USERNAME/memcached-operator-bundle:v$VERSION # location where your bundle will be hosted

make bundle
operator-sdk bundle validate ./bundle

make catalog-build catalog-push CATALOG_IMG=quay.io/zhangchl007/memcached-operator-catalog:v0.0.7

kc apply -f  deploy/olm/memcached-operator-catalogsources.yaml
kc get packagemanifest  |grep -i memcached

kc create ns olm-demo
kc -n olm-demo apply -f deploy/olm/memcached-operator-controller-manager_rbac.authorization.k8s.io_v1_clusterrolebinding.yaml
kc apply -f  deploy/olm/memcached-operator-subscription.yaml
kc -n olm-demo get og,sub,ip,csv,pods

kc apply -f config/samples/cache_v1alpha1_memcached.yaml

```
