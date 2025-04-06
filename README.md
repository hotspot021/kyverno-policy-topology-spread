Rationale:
DevOps are responsible for providing working kubernetes cluster with minimum effort for developers to deploy their applications. 
Developers write their own applications manifests (like Deployment, Services, Ingresses), but they don’t want to deal with AWS related stuff like zones and high availability. 
K8s administrator can enforce some of those rules inside the kubernetes cluster.

Task:
Implement kubernetes policies that will make sure pods for each application are balanced as much as possible across all available zones. 

Solution:
K8s policy engine like Kyverno works as custom mutating admission webhook.
At scheduling, it evaluates where to place given pod based on pre-defined rules.

Cons:
It doesn't take care of already deployed pods.
It may clash with taints or node affinity (if used).