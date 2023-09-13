# k8s-networking

Repository with notes and code created while learning Kubernetes networking

## Links

* [Academy Tigera](https://courses.academy.tigera.io/dashboard):
  * [Certified Calico Operator: Level 1](https://academy.tigera.io/course/certified-calico-operator-level-1/)
  * [Certified Calico Operator: AWS Expert](https://academy.tigera.io/course/certified-calico-operator-aws-expert/)
  * [Certified Calico Operator: Azure Expert](https://academy.tigera.io/course/certified-calico-operator-azure-expert/)
  * [Certified Calico Operator: eBPF](https://academy.tigera.io/course/certified-calico-operator-ebpf/)
* Home lab:
  * [Kind multi-node](https://docs.tigera.io/calico/latest/getting-started/kubernetes/kind)
  * [Kind quickstart](https://kind.sigs.k8s.io/docs/user/quick-start/)
  * [Kind networking](https://kind.sigs.k8s.io/docs/user/configuration/#networking)

## Certified Calico Operator: Level 1

### Multi-node kind cluster without default CNI

```
cat > multi-node-k8s-no-cni.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
EOF

kind create cluster --config multi-node-k8s-no-cni.yaml --name home-lab

kubectl get nodes -o wide

kind get clusters

kind delete cluster -n home-lab
```

### Calico

#### Installation

```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

kubectl get pods -n tigera-operator
```

#### Configuration

No encapsulation:

```
cat <<EOF | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    containerIPForwarding: Enabled
    ipPools:
    - cidr: 192.168.0.0/16
      natOutgoing: Enabled
      encapsulation: None
EOF
```

VXLAN encapsulation and API server:

```
cat <<EOF | kubectl apply -f -
# This section includes base Calico installation configuration.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

---

# This section configures the Calico API server.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF
```

#### Validation

```
kubectl get tigerastatus/calico

kubectl get pods -A

kubectl get pods -n calico-system

kubectl get nodes -A
```

#### Sample application

```
wget https://raw.githubusercontent.com/tigera/ccol1/main/yaobank.yaml

sed -i '' -e 's/node1/home-lab-worker/g' yaobank.yaml
sed -i '' -e 's/node2/home-lab-worker2/g' yaobank.yaml

kubectl apply -f yaobank.yaml

kubectl get nodes -o wid ### get IP of the node

kubectl -n yaobank get svc ### get port for the service

docker exec -it home-lab-control-plane curl 172.18.0.3:30180

CUSTOMER_POD=$(kubectl get pods -n yaobank -l app=customer -o name)

kubectl exec -it $CUSTOMER_POD -n yaobank -c customer -- /bin/bash
```

#### Sample Kubernetes network policy

Allow traffic from summary application to database:

```
cat <<EOF | kubectl apply -f -
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: database-policy
  namespace: yaobank
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: summary
    ports:
      - protocol: TCP
        port: 2379
  egress:
    - to: []
EOF
```

#### Sample Calico global network policy

Default deny:

```
cat <<EOF | calicoctl apply --allow-version-mismatch -f -
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-app-policy
spec:
  namespaceSelector: has(projectcalico.org/name) && projectcalico.org/name not in {"kube-system", "calico-system", "calico-apiserver"}
  types:
  - Ingress
  - Egress
EOF
```

Allow DNS:

```
cat <<EOF | calicoctl apply --allow-version-mismatch -f -
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-app-policy
spec:
  namespaceSelector: has(projectcalico.org/name) && projectcalico.org/name not in {"kube-system", "calico-system"}
  types:
  - Ingress
  - Egress
  egress:
    - action: Allow
      protocol: UDP
      destination:
        selector: k8s-app == "kube-dns"
        ports:
          - 53
EOF
```

```
cat <<EOF | calicoctl apply -f -
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: nodeport-policy
spec:
  order: 100
  selector: has(kubernetes.io/hostname)
  applyOnForward: true
  preDNAT: true
  ingress:
  - action: Allow
    protocol: TCP
    destination:
      ports: [30180]
    source:
      nets:
      - 198.19.15.254/32
  - action: Deny
    protocol: TCP
    destination:
      ports: ["30000:32767"]
  - action: Deny
    protocol: UDP
    destination:
      ports: ["30000:32767"]
EOF
```

#### Network policy for nodes

```
cat <<EOF| calicoctl --allow-version-mismatch apply -f -
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-node-policy
spec:
  selector: has(kubernetes.io/hostname)
  ingress:
  - action: Allow
    protocol: TCP
    source:
      nets:
      - 127.0.0.1/32
  - action: Allow
    protocol: UDP
    source:
      nets:
      - 127.0.0.1/32
EOF
```

```
calicoctl --allow-version-mismatch get heps
```

```
calicoctl --allow-version-mismatch patch kubecontrollersconfiguration default --patch='{"spec": {"controllers": {"node": {"hostEndpoint": {"autoCreate": "Enabled"}}}}}'
```

#### More Kubernetes network policies

Allow HTTP ingress and between other microservices:

```
cat <<EOF | kubectl apply -f - 
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: customer-policy
  namespace: yaobank
spec:
  podSelector:
    matchLabels:
      app: customer
  ingress:
    - ports:
      - protocol: TCP
        port: 80
  egress:
    - to: []
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: summary-policy
  namespace: yaobank
spec:
  podSelector:
    matchLabels:
      app: summary
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: customer
      ports:
      - protocol: TCP
        port: 80
  egress:
    - to:
      - podSelector:
          matchLabels:
            app: database
      ports:
      - protocol: TCP
        port: 2379
EOF
```

#### More Calico network policies

```
cat <<EOF | calicoctl apply --allow-version-mismatch -f -
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: egress-lockdown
spec:
  order: 600
  namespaceSelector: has(projectcalico.org/name) && projectcalico.org/name not in {"kube-system", "calico-system"}
  serviceAccountSelector: internet-egress not in {"allowed"}
  types:
  - Egress
  egress:
    - action: Deny
      destination:
        notNets:
          - 10.0.0.0/8
          - 172.16.0.0/12
          - 192.168.0.0/16
          - 198.18.0.0/15
EOF
```

```
kubectl label serviceaccount -n yaobank customer internet-egress=allowed
```

## Certified Calico Operator: AWS Expert

### AWS Networking

Terminology:
* AWS Global Cloud
* AWS Region
* Virtual Private Cloud (VPC)
* Availability Zone (AZ)
* Subnet
* Elastic Network Interface (ENI)

### Kubernets Cluster Options

* [Amazon Elastic Kubernetes Service (EKS)](https://docs.tigera.io/calico/latest/getting-started/kubernetes/managed-public-cloud/eks)
  * fully managed control plane
  * less expertise required
  * integrated with ECR, ELB, IAM, VPC etc.
  * Calico is built-in
  * more expensive
  * you still needs to manage EC2 nodes (unless you use AWS Fargate (Serverless compute for containers))
  * networking:
    * Amazon VPC:
      * policy - Calico
      * IPAM - AWS
      * CNI - AWS
      * Overlay - No
      * Routing - VPC native
      * Datastore - K8s
    * Calico:
      * policy - Calico
      * IPAM - Calico
      * CNI - Calico
      * Overlay - VXLAN
      * Routing - Calico
      * Datastore - K8s
* [Bring Your Own Cloud (BYOC) (using EC2)](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-public-cloud/aws)
  * flexible
  * less costly
  * e.g. [kops](https://kops.sigs.k8s.io/networking/calico/)

```
kops create cluster \
  --zones $ZONES \
  --networking calico \
  --yes \
  --name myclustername.mydns.io
```