# k8s-networking

Repository with notes and code created while learning Kubernetes networking

## Links

* [Academy Tigera](https://courses.academy.tigera.io/dashboard):
  * [Certified Calico Operator: Level 1](https://academy.tigera.io/course/certified-calico-operator-level-1/)
  * [Certified Calico Operator: eBPF](https://academy.tigera.io/course/certified-calico-operator-ebpf/)
  * [Certified Calico Operator: AWS Expert](https://academy.tigera.io/course/certified-calico-operator-aws-expert/)
  * [Certified Calico Operator: Azure Expert](https://academy.tigera.io/course/certified-calico-operator-azure-expert/)
* Home lab:
  * [Kind multi-node](https://docs.tigera.io/calico/latest/getting-started/kubernetes/kind)
  * [Kind quickstart](https://kind.sigs.k8s.io/docs/user/quick-start/)
  * [Kind networking](https://kind.sigs.k8s.io/docs/user/configuration/#networking)

## Lab

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
```

#### Sample network policy

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

### Sample Calico global network policy

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