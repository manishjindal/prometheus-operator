# RBAC

RBAC for the Prometheus Operator involves two parts, RBAC rules for the Operator itself and RBAC rules for the Prometheus Pods themselves created by the Operator as Prometheus requires access to the Kubernetes API for target and Alertmanager discovery.

## Prometheus Operator RBAC

In order for the Prometheus Operator to work in an RBAC based authorization environment, a `ClusterRole` with access to all the resources the Operator requires for the Kubernetes API needs to be created. This section is intended to describe, why the specified rules are required.

Here is a ready to use manifest of a `ClusterRole` that can be used to start the Prometheus Operator:

[embedmd]:# (../example/rbac/prometheus-operator-cluster-role.yaml)
```yaml
apiVersion: rbac.authorization.k8s.io/v1alpha1
kind: ClusterRole
metadata:
  name: prometheus-operator
rules:
- apiGroups:
  - extensions
  resources:
  - thirdpartyresources
  verbs:
  - create
- apiGroups:
  - monitoring.coreos.com
  resources:
  - alertmanagers
  - prometheuses
  - servicemonitors
  verbs:
  - "*"
- apiGroups:
  - apps
  resources:
  - statefulsets
  verbs: ["*"]
- apiGroups: [""]
  resources:
  - configmaps
  - secrets
  verbs: ["*"]
- apiGroups: [""]
  resources:
  - pods
  verbs: ["list", "delete"]
- apiGroups: [""]
  resources:
  - services
  - endpoints
  verbs: ["get", "create", "update"]
- apiGroups: [""]
  resources:
  - nodes
  verbs: ["list", "watch"]
```

> Note: A cluster admin is required to create this `ClusterRole` and create a `ClusterRoleBinding` or `RoleBinding` to the `ServiceAccount` used by the Prometheus Operator `Pod`. The `ServiceAccount` used by the Prometheus Operator `Pod` can be specified in the `Deployment` object used to deploy it.

When the Prometheus Operator boots up for the first time it registers the `thirdpartyresources` it uses, therefore the `create` action on those is required.

As the Prometheus Operator work extensively with the `thirdpartyresources` it registers, it requires all actions on those objects. Those are:

* `alertmanagers`
* `prometheuses`
* `servicemonitors`

Alertmanager and Prometheus clusters are created using `statefulsets` therefore all changes to an Alertmanager or Prometheus object result in a change to the `statefulsets`, which means all actions must be permitted.

Additionally as the Prometheus Operator takes care of generating configurations for Prometheus to run, it requires all actions on `configmaps`.

When the Prometheus Operator performs version migrations from one version of Prometheus or Alertmanager to the other it needs to `list` `pods` running an old version and `delete` those.

The Prometheus Operator reconciles `services` called `prometheus-operated` and `alertmanager-operated`, which are used as governing `Service`s for the `StatefulSet`s. To perform this reconciliation

As the kubelet is currently not self-hosted, the Prometheus Operator has a feature to synchronize the IPs of the kubelets into an `Endpoints` object, which requires access to `list` and `watch` of `nodes` (kubelets) and `create` and `update` for `endpoints`.

## Prometheus RBAC

The Prometheus server itself accesses the Kubernetes API to discover targets and Alertmanagers. Therefore a separate `ClusterRole` for those Prometheus servers need to exist.

As Prometheus does not modify any Objects in the Kubernetes API, but just reads them it simply requires the `get`, `list`, and `watch` actions.

[embedmd]:# (../example/rbac/prometheus-cluster-role.yaml)
```yaml
apiVersion: rbac.authorization.k8s.io/v1alpha1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
```

> Note: A cluster admin is required to create this `ClusterRole` and create a `ClusterRoleBinding` or `RoleBinding` to the `ServiceAccount` used by the Prometheus `Pod`s.  The `ServiceAccount` used by the Prometheus `Pod`s can be specified in the `Prometheus` object.

## Example

To demonstrate how to use a `ClusterRole` with a `ClusterRoleBinding` and a `ServiceAccount` here an example. It is assumed, that both of the `ClusterRole`s described above are already created.

Say the Prometheus Operator shall be deployed in the `default` namespace. First a `ServiceAccount` needs to be setup.

[embedmd]:# (../example/rbac/prometheus-operator-service-account.yaml)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-operator
```

Note that the `ServiceAccountName` also has to actually be used in the `PodTemplate` of the `Deployment` of the Prometheus Operator.

And then a `ClusterRoleBinding`:

[embedmd]:# (../example/rbac/prometheus-operator-cluster-role-binding.yaml)
```yaml
apiVersion: rbac.authorization.k8s.io/v1alpha1
kind: ClusterRoleBinding
metadata:
  name: prometheus-operator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-operator
subjects:
- kind: ServiceAccount
  name: prometheus-operator
  namespace: default
```

Because the `Pod` that the Prometheus Operator is running in uses the `ServiceAccount` named `prometheus-operator` and the `RoleBinding` associates it with the `ClusterRole` named `prometheus-operator` it now has permission to access all the resources as described above.

When creating `Prometheus` objects the procedure is similar. It starts with a `ServiceAccount`.

[embedmd]:# (../example/rbac/prometheus-service-account.yaml)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
```

And then because the `ClusterRole` named `prometheus`, as described above, is likely to be used multiple times, a `ClusterRoleBinding` instead of a `RoleBinding` is used.

[embedmd]:# (../example/rbac/prometheus-cluster-role-binding.yaml)
```yaml
apiVersion: rbac.authorization.k8s.io/v1alpha1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: default
```

> See [Using Authorization Plugins](https://kubernetes.io/docs/admin/authorization/) for further usage information on RBAC components.