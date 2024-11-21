# AWS EKS Networking

## Amazon VPC CNI

Pod networking, also called cluster networking, is the center of Kuberntes netwroking. Kubernetes supports Container Network Interface (CNI) plugins for clustering networking.

Amazon EKS uses Amazon VPC to provide networking capabilities to worker nodes and Kubernetes Pods. An EKS cluster consists of two VPCs: and AWS managed VPC that hosts the Kubernetes contorl plane and a second customer-managed VPC that hosts the Kuberenetes worker nodes where container run , as well as other AWS infrastructure used by a clusater. All worker ndoes need the abitlit to conenect to the managesd API server endpoint.
* Worker nodes connect to the EKS control plane through the EKS public endpoint or EKS-managed elastic network interfaces (ENIs).
* The subnets that you pass when you create a cluster infulence where EKS places these ENIs.
* You need to provide __minimum two subnets in two different availability zones__.
* The route that worker nodes take to connect is determined by whether you have enabled or disabled the private endpoint for your cluster.
* EKS uses the EKS managed ENI to communicate with worker nodes
* Amazon officially supports VCP CNI Plugin to implement Kubernetes Pod Networking.
* The VPC CNI provides native integration with AWS VPC and works in underlay mode.
* Pods and hosts are located at the same network layer and share the network namespace.
* The IP address of the Pod consistent from the cluster and VPC perspective.
* By default Kubernetes allows all pods to freely communicate with each other with no restrictions.

The network policy specification contains the following key segments:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - ipBlock:
            cidr: 172.17.0.0/16
            except:
              - 172.17.1.0/24
        - namespaceSelector:
            matchLabels:
              project: myproject
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 6379
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/24
      ports:
        - protocol: TCP
          port: 5978
```
__metadata__: similar to other Kubernetes objects, it allows you to specify the name and namespace for the given network policy.
spec.podSelector: allows for the selection of specific pods based on their labels within the namespace to which the given network policy will be applied. If an empty pod selector or matchLabels is specified in the specification, then the policy will be applied to all the pods within the namespace.
__spec.policyTypes__: specifies whether the policy will be applied to ingress traffic, egress traffic, or both for the selected pods. If you do not specify this field, then the default behavior is to apply the network policy to ingress traffic only, unless the network policy has an egress section, in which case the network policy will be applied to both ingress and egress traffic.
__ingress__: allows for ingress rules to be configured that specify from which pods (podSelector), namespace (namespaceSelector), or CIDR range (ipBlock) traffic is allowed to the selected pods and which port or port range can be used. If a port or port range is not specified, any port can be used for communication.


__For more information about what capabilities are allowed or restricted for Kubernetes network policies, refer to the Kubernetes docs.__


* In addition to network policies, Amazon VPC CNI in IPv4 mode offers a powerful feature known as "Security Groups for Pods."
  * Security groups allow control of ingress and egress traffic to CIDR ranges, whereas network policies allow control of ingress and egress traffic to pods, namespaces as well as CIDR ranges.
  * Security groups allow control of ingress and egress traffic from other security groups, which is not available for network policies.
* Amazon EKS strongly recommends employing network policies in conjunction with security groups to restrict network communication between pods, thus reducing the attack surface and minimizing potential vulnerabilities.


* __Default Deny__ policy applied to all namespaces.

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
    - Egress
```

* Implementing the above policy will also cause the sample application to no longer function properly as 'ui' component requires access to the 'catalog' service and other service components. To define an effective egress policy for 'ui' component requires understanding the network dependencies for the component.
* The below network policy was designed considering the above requirements. It has two key sections:
  * The first section focuses on allowing egress traffic to all service components such as 'catalog', 'orders' etc. without providing access to the database components through a combination of namespaceSelector, which allows for egress traffic to any namespace as long as the pod labels match "app.kubernetes.io/component: service".
  * The second section focuses on allowing egress traffic to all components in the kube-system namespace, which enables DNS lookups and other key communications with the components in the system namespace.

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
namespace: ui
name: allow-ui-egress
spec:
podSelector:
matchLabels:
app.kubernetes.io/name: ui
policyTypes:
- Egress
egress:
- to:
- namespaceSelector:
matchLabels:
podSelector:
matchLabels:
app.kubernetes.io/component: service
- namespaceSelector:
matchLabels:
kubernetes.io/metadata.name: kube-system
```

