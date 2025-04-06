# Rationale
DevOps are responsible for providing working kubernetes cluster with minimum effort for developers to deploy their applications. 
Developers write their own applications manifests (like Deployment, Services, Ingresses), but they don’t want to deal with AWS related stuff like zones and high availability. 
K8s administrator can enforce some of those rules inside the kubernetes cluster.

# Task
Implement kubernetes policies that will make sure pods for each application are balanced as much as possible across all available zones. 

# Solution
K8s policy engine like Kyverno works as custom mutating admission webhook.
At scheduling, it evaluates where to place given pod based on pre-defined rules.

# Cons
It doesn't take care of already deployed pods.
It may clash with taints or node affinity (if used).

# How to use
### Pre-req
#### 1/

K8s cluster with multiple nodes, each node labeled with keys
- kubernetes.io/hostname
- zone

For example for 6 worker node cluster it may look like:

```
nasko@D7440: /mnt/c/projects/kubernetes/kyverno-policy-topology-spread (main)
# k get nodes --show-labels
NAME                 STATUS   ROLES           AGE     VERSION   LABELS
kind-control-plane   Ready    control-plane   6h28m   v1.32.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-control-plane,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
kind-worker          Ready    <none>          6h28m   v1.32.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker,kubernetes.io/os=linux,zone=A
kind-worker2         Ready    <none>          6h28m   v1.32.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker2,kubernetes.io/os=linux,zone=B
kind-worker3         Ready    <none>          6h28m   v1.32.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker3,kubernetes.io/os=linux,zone=C
kind-worker4         Ready    <none>          6h28m   v1.32.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker4,kubernetes.io/os=linux,zone=A
kind-worker5         Ready    <none>          6h28m   v1.32.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker5,kubernetes.io/os=linux,zone=B
kind-worker6         Ready    <none>          6h28m   v1.32.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker6,kubernetes.io/os=linux,zone=C
nasko@D7440: /mnt/c/projects/kubernetes/kyverno-policy-topology-spread (main)

```

#### 2/
[Kyverno](https://kyverno.io/docs/installation/methods/) policy engine already installed inside the k8s cluster

### Usage

Then just apply the policy.

> kubectl apply -f clusterpolicy-admission-distribute-pods.yml

Test

> kubectl describe clusterpolicy/spread-pods-across-zones-and-nodes
> kubectl get clusterpolicy/spread-pods-across-zones-and-nodes -o yaml

Create simple deployment and observe the mutation (auto addition) of topologySpreadConstraints

```
nasko@D7440: /mnt/c/projects/kubernetes/kyverno-policy-topology-spread (main)
# kubectl create deployment my-deployment --image=nginx --replicas=5
deployment.apps/my-deployment created
nasko@D7440: /mnt/c/projects/kubernetes/kyverno-policy-topology-spread (main)
# k get pods -o wide |grep my-deployment
my-deployment-5bc86f79c6-4g9tb   1/1     Running   0          29s    10.244.6.6   kind-worker2   <none>           <none>
my-deployment-5bc86f79c6-7qmvq   1/1     Running   0          29s    10.244.1.6   kind-worker5   <none>           <none>
my-deployment-5bc86f79c6-9rm9p   1/1     Running   0          29s    10.244.4.6   kind-worker3   <none>           <none>
my-deployment-5bc86f79c6-cw6lf   1/1     Running   0          29s    10.244.3.6   kind-worker    <none>           <none>
my-deployment-5bc86f79c6-m9llj   1/1     Running   0          29s    10.244.5.6   kind-worker4   <none>           <none>
nasko@D7440: /mnt/c/projects/kubernetes/kyverno-policy-topology-spread (main)

```

And the `topologySpreadConstraints` is auto added by the mutation admission kyverno policy
 
```
nasko@D7440: /mnt/c/projects/kubernetes/kyverno-policy-topology-spread (main)
# k get pod/my-deployment-5bc86f79c6-4g9tb -o yaml |grep topologySpreadConstraints -A12
  topologySpreadConstraints:
  - labelSelector:
      matchLabels:
        app: my-deployment
    maxSkew: 1
    topologyKey: zone
    whenUnsatisfiable: ScheduleAnyway
  - labelSelector:
      matchLabels:
        app: my-deployment
    maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
nasko@D7440: /mnt/c/projects/kubernetes/kyverno-policy-topology-spread (main)

```