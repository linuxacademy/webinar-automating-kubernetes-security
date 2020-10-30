# Automating Kubernetes Security

This is a walkthrough guide for the live demo performed during the `Automating Kubernetes Security` webinar.

## Cluster Setup
Set up a Kubernetes cluster with `kubeadm`, or use an existing one.

Demo environment info:
- OS: `Ubuntu 18.04 Bionic Beaver LTS`
- Container Runtime: `Docker 19.03.12`
- Kubernetes version: `1.19.2`
- Networking Plugin: `Calico`

## Set Up a Namespace and Non-Admin ServiceAccount to Use for Testing
Create a Namespace.

```
kubectl create ns development
```

Create the ServiceAccount.

```
kubectl create sa nonadmin -n development
```

Create a RoleBinding allowing the account to edit objects in the `development` Namespace.

```
vi rb-nonadmin-edit.yml
```

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rb-nonadmin-edit
  namespace: development
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: nonadmin
  namespace: development
```

```
kubectl create -f rb-nonadmin-edit.yml --save-config
```

Verify that you can run commands as the `nonadmin` user. You should see the message `No resources found in development namespace.` since no Pods have been created in the Namespace.

```
kubectl --as=system:serviceaccount:development:nonadmin get pods -n development
```

## Create a PodSecurityPolicy
Create a new PodSecurityPolicy that will prevent pods from using privileged mode.

```
vi psp-nopriv.yml
```

```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp-nopriv
spec:
  privileged: false
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - secret
  - projected
```

Create the PodSecurityPolicy.

```
kubectl create -f psp-nopriv.yml --save-config
```

## Set Up RBAC to Authorize the Use of the PodSecurityPolicy for a New ServiceAccount
Create a ClusterRole that allows the use of our PodSecurityPolicy.

```
vi cr-use-psp.yml
```

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cr-use-psp
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - psp-nopriv
```

Create the ClusterRole.

```
kubectl create -f cr-use-psp.yml --save-config
```

Create a ServiceAccount.

```
kubectl create sa nonprivileged -n development
```

Create a RoleBinding that allows the `nonprivileged` ServiceAccount to use the PodSecurityPolicy in the `default` Namespace.

```
vi rb-nonprivileged.yml
```

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rb-nonprivileged
  namespace: development
roleRef:
  kind: ClusterRole
  name: cr-use-psp
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: nonprivileged
  namespace: development
```

Create the RoleBinding.

```
kubectl create -f rb-nonprivileged.yml --save-config
```

## Turn on the PodSecurityPolicy Admission Controller

```
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Locate the line for the `--enable-admission-plugins` and add `PodSecurityPolicy` to the list.

```
- --enable-admission-plugins=NodeRestriction,PodSecurityPolicy
```

Run a command to make sure the API Server is still responding after the change to the manifest file (it may take a few moments to come back up).

```
kubectl get nodes
```

## Create a Pod
Create a basic Pod, using the `nonadmin` ServiceAccount created earlier.

```
vi pod-basic.yml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-basic
  namespace: development
spec:
  containers:
  - name: nginx
    image: nginx
```

```
kubectl create -f pod-basic.yml --as=system:serviceaccount:development:nonadmin --save-config
```

This operation will fail, since the Pod does not have access to any PodSecurityPolicies that would allow it to be created.

Let's try again, this time ensuring the Pod has access to our PodSecurityPolicy by giving it the `nonprivileged` ServiceAccount.

```
vi pod-with-psp-access.yml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-psp-access
  namespace: development
spec:
  serviceAccountName: nonprivileged
  containers:
  - name: nginx
    image: nginx
```

```
kubectl create -f pod-with-psp-access.yml --as=system:serviceaccount:development:nonadmin --save-config
```

This time, it should succeed since the Pod's ServiceAccount has access to the PodSecurityPolicy, and the Pod does not violate any policies.

Examine the Pod with `kubectl describe`.

```
kubectl describe pod pod-with-psp-access -n development
```

Note that there an annotation which lists the PodSecurityPolicy that allowed the Pod.

Let's try to create a Pod that does violate the policy by requesting privileged access.

```
vi pod-privileged.yml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-privileged
  namespace: development
spec:
  serviceAccountName: nonprivileged
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true
```

```
kubectl create -f pod-privileged.yml --as=system:serviceaccount:development:nonadmin --save-config
```

This should fail, since the privileged mode containers are not allowed by the PodSecurityPolicy.

## Fix Mirror Pod Creation for Static Pods
List your system Pods.

```
kubectl get pods -n kube-system
```

You may notice that the kube-apiserver mirror Pod is missing. While the API Server itself is in fact running since it is managed by kubelet and bypasses PodSecurityPolicies, the PodSecurityPolicies are preventing the mirror Pod from being created. This makes these system Pods invisible in the Kubernetes API. Let's fix it!

Create a new PodSecurityPolicy that will allow static mirror Pods.

```
vi psp-static.yml
```

```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp-static
spec:
  allowPrivilegeEscalation: true
  fsGroup:
    rule: RunAsAny
  hostNetwork: true
  runAsUser:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - secret
  - projected
  - hostPath
```

```
kubectl create -f psp-static.yml --save-config
```

Create a ClusterRole with access to the policy.

```
vi cr-use-psp-static.yml
```

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cr-use-psp-static
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - psp-static
```

```
kubectl create -f cr-use-psp-static.yml --save-config
```

Create a ClusterRoleBinding to allow Kubernetes Nodes to use the policy.

```
vi crb-use-psp-static.yml
```

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: crb-use-psp-static
roleRef:
  kind: ClusterRole
  name: cr-use-psp-static
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: Group
  name: system:nodes
```

```
kubectl create -f crb-use-psp-static.yml --save-config
```

List system Pods again.

```
kubectl get pods -n kube-system
```

You should see a `kube-apiserver` Pod appear. If it doesn't, you may need to wait a minute or two and try again.
