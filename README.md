# memcached-operator-demo

go version 1.17

operator-sdk v1.19.1


# operator ops 
```
operator-sdk init --domain example.com --repo github.com/example/memcached-operator

operator-sdk create api --group cache --version v1alpha1 --kind Memcached --resource --controller

# modify api and controller


# generate manifest

make generate
make manifests
make undeploy

#make operator docker image

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
