---
layout: post
comments: true
title: "Pod Security Policies on Kubernetes"
fulltitle: ""
excerpt: ""
categories : 
- howto
---

This is a quick guide, or mainly additional notes, around implementing Pod Security Policies on Kubernetes.
The official guide (https://kubernetes.io/docs/concepts/policy/pod-security-policy/#create-a-policy-and-a-pod) provides an example but does not explain all the things, such as why do things in a particular order, and why the options chosen were chosen.
This post aims to fill in those gaps and serve as a reminder next time.

## 1. Enable the PodSecurityPolicy admission controller.

In kube, admission controllers validate requests to the API after authz+authn has occured.
The PodSecurityPolicy specifically acts on creation and modification of pods.

Enable it by adding `PodSecurityPolicy` to the `--admission-control` arg of `kube-apiserver` daemon, for me this is in `/etc/kubernetes/manifests/kube-apiserver.yaml`.

Initially, as no security policy exists, pod creation will be denied for all but the admin (or whoever is using `/etc/kubernetes/admin.conf` with their client).

## 2. Create a service account (and namespace if required)

Create a namespace to hold things that should be together, together.
If there is already an existing namespace that you want to use, then skip that.

```
kubectl create namespace <namespace>
```

Then create a service account within that namespace that this policy will be applied to.

```
kubectl create serviceaccount -n <namespace> <username>
```

Again, if there is already an existing service account within an existing namespace that you want to use, it is not necessary to create them again.

## 3. Bind a role to the new service account

By default, kube includes 3 roles:

- `admin`: read/write on everything within the namespace
- `edit`: read/write on the namespace except modifying roles or role bindings
- `view`: read on the namespace except secrets

The role binding needs a name, `<rolebindingname>`, and I suggest using something like `<username>:<role>`.

```
kubectl create rolebinding -n <namespace> <rolebindingname> --clusterrole=<role> --serviceaccount=<namespace>:<username>
```

## 4. Create a security policy if an appropriate one doesn't already exist

List existing policies with

```
kubectl get psp
```

Otherwise, create one.

For example, this policy blocks privileged mode, where `unprivileged` is the `<policyname>`.

```
kubectl -n <namespace> create -t - <<EOF
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
metadata:
  name: unprivileged
spec:
  privileged: false  # Don't allow privileged pods!
  # The rest fills in some required fields.
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
EOF
```

When creating a new policy, also create a role which will allow it to be bound (via a role binding) to a particular user. I suggest using the format `psp:<policyname>` for the role name.
The `use` verb authorises the usage of this policy - this is important because by default everything is blocked and policies can only allow things to take place.

```
kubectl -n <namespace> create role <policyrolename> \
    --verb=use \
    --resource=podsecuritypolicy \
    --resource-name=<policyname>
```

For this example, that would be:

```
kubectl -n <namespace> create role psp:unprivileged \
    --verb=use \
    --resource=podsecuritypolicy \
    --resource-name=unprivileged
```

Also ensure that you create a role binding for the default pod service account in the namespace (account used by the deployment controllers).

```
kubectl -n <namespace> create rolebinding default:<policyrolename> \
    --role=<policyrolename> \
    --serviceaccount=<namespace>:default
```

For this example, that would be:

```
kubectl -n <namespace> create rolebinding default:psp:unprivileged \
    --role=psp:unprivileged \
    --serviceaccount=<namespace>:default
```

## 5. Bind the security policy role to the user

```
kubectl -n <namespace> create rolebinding <username>:<policyrolename> \
    --role=<policyrolename> \
    --serviceaccount=<namespace>:<username>
```

Check that the user can use this new role

```
kubectl --as=system:serviceaccount:<namespace>:<username> -n <namespace> auth can-i use podsecuritypolicy/<policyname>
```

## 6. Test your limits!

```
kubectl --as=system:serviceaccount:<namespace>:<username> -n <namespace> create -f- <<EOF
apiVersion: v1
kind: Pod
metadata:
  name:      privileged
spec:
  containers:
    - name:  pause
      image: k8s.gcr.io/pause
      securityContext:
        privileged: true
EOF
```
