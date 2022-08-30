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

