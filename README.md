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

### Kubernetes Networking Options

* K8s with Amazon VPC CNI Networking (AWS-CNI)
  * VPC exists in a region
  * VPCs span AZs
  * subnet reside in 1 AZ
  * VPC can contain secondary subnets
  * max subnet is /16
  * max 5 CIDR block associate with VPC
  * each pod and node requires IP
  * IP address exhaustion is a consideration
  * each compute node can have limited ENIs and IPs e.g.
    * `t3.2xlarge` max IP = 15, max ENI = 4
    * `t3.small` max IP = 4, max ENI = 3
  * IPs are known by the fabric (routing from outside and from EKS control nodes just works)
  * AWS-CNI spreads pods over multiple ENIs, so we have more bandwidth
  * IAM integration can be tight
* K8s with Calico CNI
  * both EKS and self-managed clusters can deploy Calico CNI networking
  * non-native pod IPs are used
  * more scalable
  * IP addresss exhaustion is no longer a concern
  * high performance VXLAN tunnels to allow cluster nodes to communicate
  * VXLAN encapsulation: Ethernet header -> IP header -> UDP header -> VXLAN header -> original packet -> Ethernet footer
  * in self-managed clusters Calico support 3 modes:
    * IP-IP (tunnel mode)
    * VXLAN (tunnel mode)
    * CrossSubnet (hybrid mode)
  * connections from pod to outside the cluster needs to Source NATed
  * if using CrossSubnet overlay mode, the use needs to disable src/dst check
  * control plane nodes will not be able to initiate network connections to Calico pods without workaround

* AWS-CNI > Calico CNI:
  * benfit from tighter integration with other AWS features
* Calico-CNI > AWS-CNI
  * no limit to number of pods per node
  * encryption

### Kubernetes Network Security

* Security groups
  * EKS
    * EKS cluster create a cluster security group
    * traffic from control plane and managed node groups can flow freely
    * EKS supports assigning security groups to pods
      * some instance types are not supported
      * SG association means only SG - no Calico network policy enforcement
      * source NAT is disabled, so Internet access is complex (private subnet with NAT gateway required)
  * BYOC
    * security groups without EKS do not have pod label visibility
    * they are only suitable for coarse-grained network policy
* Network policies (EKS & BYOC)
  * primary tool for securing K8s
  * K8s network model is "flat" network in which every pod can communicate by IP
  * simplified network design
  * network policies are abstracted from the network by using label selectors
* Encryption (EKS & BYOC)
  * without Calico CNI service mesh is needed to encrypt data
  * Calico CNI offers WireGuard encryption to protect data in flight
  * one-liner to enable

### Exposing Services Outside the Cluster

* Pods are ephemeral -> IPs change rapidly
* Need for predictiable service
* K8s services:
  * NodePort
    * TCP/UDP port on cluster node
    * traffic forwarded to pod (based on label)
    * find nodes by DNS
    * source and destination NAT
    * externalTrafficPolicy: local or eBPF for source IP preservation
  * LoadBalancer (L4)
    * for BYOC can vary implementation
      * source and destination NAT
      * externalTrafficPolicy: local or eBPF for source IP preservation
    * for EKS it's based on cloud provider LB
      * NodePort and ClusterIp service automatically created
      * AWS LB:
        * ELB (classic) (don't have source IP preservation)
        * ALB (L7)
        * NLB (L4)
  * Ingress (L7, not technically service)
    * gateway that manages external access to HTTP(s) service in cluster
    * considered as proxy
    * requires ingress controller (in BYOC) or cloud implementation (in EKS)
* [GitHub - Certified Calico Operator - L2 - AWS Expert](https://github.com/tigera/ccol2aws)

### EKS Deployment

* [EKS Deployment Options](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html):
  * GUI
  * [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-eks-cluster.html) / [Terraform](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest)
  * [Quickstart](https://aws-quickstart.github.io/quickstart-eks-tigera-calico/)
  * CLI:
    * [List EKS clusters](https://docs.aws.amazon.com/cli/latest/reference/eks/list-clusters.html)
    * [Update a kubeconfig](https://docs.aws.amazon.com/cli/latest/reference/eks/update-kubeconfig.html)
  * [`eksctl`](https://eksctl.io/)
* Installing Calico:
  * [Install EKS with Amazon VPC networking and Calico network policy engine add-on](https://docs.aws.amazon.com/eks/latest/userguide/calico.html)
  * [Install EKS with Calico networking](https://docs.tigera.io/calico/latest/getting-started/kubernetes/managed-public-cloud/eks)

### EKS Lab with AWS-CNI

Cluster deployment:

```
curl -L https://raw.githubusercontent.com/tigera/ccol2aws/main/bootstrap.sh | sh
. ccol2awsexports.sh
eksctl create cluster --name calicopolicy --version 1.22 --ssh-access --node-type t3.medium
kubectl get nodes -A
```

Application deployment:

```
kubectl apply -f https://raw.githubusercontent.com/tigera/ccol2aws/main/yaobank.yaml
kubectl get pods -n yaobank
```

IP address allocation:

```
aws ec2 describe-instance-types --filters "Name=instance-type,Values=t3.*" --query "InstanceTypes[].{Type: InstanceType, MaxENI: NetworkInfo.MaximumNetworkInterfaces, IPv4addr: NetworkInfo.Ipv4AddressesPerInterface}" --output table
```

```
--------------------------------------
|        DescribeInstanceTypes       |
+----------+----------+--------------+
| IPv4addr | MaxENI   |    Type      |
+----------+----------+--------------+
|  15      |  4       |  t3.2xlarge  |
|  12      |  3       |  t3.large    |
|  15      |  4       |  t3.xlarge   |
|  6       |  3       |  t3.medium   |
|  2       |  2       |  t3.nano     |
|  2       |  2       |  t3.micro    |
|  4       |  3       |  t3.small    |
+----------+----------+--------------+
```

Maximum number of pods that can be run on VM:

```
((MaxENI * (IPv4addr-1)) + 2)
```

Scale deployment:

```
kubectl scale -n yaobank --replicas 30 deployments/customer
kubectl get pods -A | grep Running | wc -l
kubectl get pods -A | grep Pending
```

Not all pods are running, because using the AWS-CNI provider for EKS has limited us to running 17 pods per node.

Inspect worker node:

```
kubectl get nodes -o wide
ssh ec2-user@1.2.3.4

ip rule
sudo iptables -S -t mangle
ip route show table main
ip route show table 2
```

### Securing application with Calico Policy

Install Calico Policy:

```
kubectl create -f https://docs.projectcalico.org/archive/v3.21/manifests/tigera-operator.yaml

kubectl apply -f -<<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  kubernetesProvider: EKS
  cni:
    type: AmazonVPC
  calicoNetwork:
    nodeAddressAutodetectionV4:
      canReach: 1.1.1.1
    bgp: Disabled
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF

kubectl get tigerastatus
```

Connectivity tests:

```
export CUSTOMER_POD=$(kubectl get pods -n yaobank -l app=customer -o name)
export SUMMARY_POD=$(kubectl get pods -n yaobank -l app=summary -o name | head -n 1)
echo "export CUSTOMER_POD=${CUSTOMER_POD}" >> ccol2awsexports.sh
echo "export SUMMARY_POD=${SUMMARY_POD}" >> ccol2awsexports.sh
chmod 700 ccol2awsexports.sh
. ccol2awsexports.sh

kubectl exec -it $CUSTOMER_POD -n yaobank -c customer -- /bin/bash
curl http://database:2379/v2/keys?recursive=true | python -m json.tool
```

Adding Global Default Deny:

```
cat <<EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-app-policy
spec:
  namespaceSelector: has(projectcalico.org/name) && projectcalico.org/name not in {"kube-system", "calico-system","calico-apiserver"}
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

kubectl exec -it $CUSTOMER_POD -n yaobank -c customer -- sh -c 'curl --connect-timeout 3 http://database:2379/v2/keys?recursive=true | python -m json.tool'
```

Apply policy:

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

kubectl exec -it $SUMMARY_POD -n yaobank -c summary -- sh -c 'curl --connect-timeout 3 http://database:2379/v2/keys?recursive=true | python -m json.tool'
```

### Deploying an Elastic Load Balancer

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: yaobank-customer
  namespace: yaobank
spec:
  selector:
    app: customer
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
EOF

kubectl get svc -n yaobank

kubectl logs -n yaobank $CUSTOMER_POD

kubectl delete service yaobank-customer -n=yaobank
```

### Lab cleamup

```
eksctl delete cluster --name calicopolicy
```

### The AWS CNI with Calico eBPF

Options in EKS:
* AWS-CNI without Calico
* AWS-CNI & Calico for Policy (`iptables`)
* AWS-CNI & Calico for Policy (`eBPF`)
* Calico CNI & Calico for Policy (`iptables`)
* Calico CNI & Calico for Policy (`eBPF`)

Pros AWS-CNI:
* pods get native IPs
* routing from outside or control nodes just works
* using multiple ENIs gives access to more bandwidth
* IAM integration is improved

Cons AWS-CNI:
* number of pods per node is limited by number of ENIs - leading to systemic underutilization

[Bottlerocket](https://aws.amazon.com/bottlerocket/):
* open source Linux distribution
* built by Amazon to run containers

`iptables` and `eBPF`:
* `iptables` works great for most users
* `eBPF` scales to higher throughput and uses less CPU per GBit
* `eBPF` with AWS-CNI in EKS replaces `kube-proxy` and reduces latency

Use Calico `eBPF` mode with AWS-CNI in EKS:
* create a cluster with an AMI that supports `eBPF`
* install Calico in `iptables` mode
* adjust Calico to talk to the K8s API directly
* disable kube-proxy
* enable `eBPF` on Felix (the Calico agent on each K8s node)

Links:
* [Turbocharging EKS networking with Bottlerocket, Calico, and eBP](https://aws.amazon.com/blogs/containers/turbocharging-eks-networking-with-bottlerocket-calico-and-ebpf/)
* [EKS, Bottlerocket, and Calico eBPF](https://www.tigera.io/blog/eks-bottlerocket-and-calico-ebpf/)
* [EKS Unchained with eBPF and Bottlerocket](https://miles-seth.medium.com/eks-unchained-with-ebpf-and-bottlerocket-1639b011a36a)

### Deploying EKS with Calico eBPF

Creating an EKS cluster with Bottlerocket:

```
curl -L https://raw.githubusercontent.com/tigera/ccol2aws/main/bootstrap.sh | sh
. ccol2awsexports.sh
eksctl create cluster --name calicoebpf --version 1.22 --ssh-access --node-type t3.medium --node-ami-family Bottlerocket
kubectl get nodes -A
```

Installing Calico:

```
kubectl create -f https://docs.projectcalico.org/archive/v3.21/manifests/tigera-operator.yaml

kubectl apply -f -<<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  kubernetesProvider: EKS
  cni:
    type: AmazonVPC
  calicoNetwork:
    nodeAddressAutodetectionV4:
      canReach: 1.1.1.1
    bgp: Disabled
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF

kubectl get tigerastatus
```

Enabling the eBPF Dataplane:

```
kubectl get configmap -n kube-system kube-proxy -o jsonpath='{.data.kubeconfig}' | grep server

cat <<EOF | kubectl apply -f -
kind: ConfigMap
apiVersion: v1
metadata:
  name: kubernetes-services-endpoint
  namespace: kube-system
data:
  KUBERNETES_SERVICE_HOST: "<SERVER_HOSTNAME_FROM_K_GET_CONFIGMAP_GREP_SERVER>"
  KUBERNETES_SERVICE_PORT: "443"
EOF

calicoctl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: vpc-subnet
spec:
  cidr: 192.168.0.0/16
  natOutgoing: true
  nodeSelector: !all()
EOF

kubectl rollout restart ds/calico-node -n calico-system
kubectl rollout restart deployment/calico-kube-controllers -n calico-system
kubectl rollout restart deployment/calico-typha -n calico-system

kubectl get pods -n kube-system
```

Replace kube-proxy and enable eBPF on Felix:

```
kubectl patch ds -n kube-system kube-proxy -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": "true"}}}}}'

calicoctl patch felixconfiguration default --patch='{"spec": {"bpfEnabled": true}}'

kubectl delete pod -n kube-system -l k8s-app=kube-dns
kubectl get pods -n kube-system | grep coredns
```

Application deployment:

```
kubectl apply -f https://raw.githubusercontent.com/tigera/ccol2aws/main/yaobank.yaml

export CUSTOMER_POD2=$(kubectl get pods -n yaobank -l app=customer -o name)
export SUMMARY_POD2=$(kubectl get pods -n yaobank -l app=summary -o name | head -n 1)
echo "export CUSTOMER_POD2=${CUSTOMER_POD2}" >> ccol2awsexports.sh
echo "export SUMMARY_POD2=${SUMMARY_POD2}" >> ccol2awsexports.sh
```

Deploying an Elastic Load Balancer:

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: yaobank-customer
  namespace: yaobank
spec:
  selector:
    app: customer
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
EOF

kubectl get svc -n yaobank yaobank-customer
curl <EXTERNAL-IP-FOR-SVC>
kubectl logs -n yaobank $CUSTOMER_POD2
kubectl delete service yaobank-customer -n=yaobank
```

Enabling Source IP Preservation with eBPF and an NLB:

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: yaobank-customer
  namespace: yaobank
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  selector:
    app: customer
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
EOF

kubectl get svc -n yaobank yaobank-customer
curl <EXTERNAL-IP-FOR-SVC>
kubectl logs -n yaobank $CUSTOMER_POD2
kubectl delete service yaobank-customer -n=yaobank
```

Cleanup:

```
eksctl delete cluster --name calicoebpf
```

### Deploying EKS with Calico CNI

Deploy our cluster without a nodegroup:

```
eksctl create cluster --name calicocni --without-nodegroup --version 1.22

kubectl get nodes -A
kubectl get pods -A
```

Delete the aws-node CNI daemonset as we will be deploying CNI nodes using Calico. The aws-node CNI daemonset would have deployed the AWS-CNI pod on each node when they were brought into the cluster:

```
kubectl delete daemonset -n kube-system aws-node
```

Install the operator:

```
kubectl create -f https://docs.projectcalico.org/archive/v3.21/manifests/tigera-operator.yaml
```

Prepare Calico configuration:

```
kubectl apply -f -<<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  kubernetesProvider: EKS
  cni:
    type: AmazonVPC
  calicoNetwork:
    nodeAddressAutodetectionV4:
      canReach: 1.1.1.1
    bgp: Disabled
    ipPools:
    - blockSize: 26
      cidr: 172.16.0.0/16
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer 
metadata: 
  name: default 
spec: {}
EOF

kubectl get tigerastatus
```

Create nodegroup:

```
eksctl create nodegroup --cluster calicocni --node-type t3.medium --node-ami-family Bottlerocket --max-pods-per-node 100 --ssh-access

kubectl get nodes -A
kubectl get pods -A
```

Deploy and scale application:

```
kubectl apply -f https://raw.githubusercontent.com/tigera/ccol2aws/main/yaobank.yaml
kubectl scale -n yaobank --replicas 30 deployments/customer
kubectl get pods -n yaobank
```

Review IP address allocation:

```
calicoctl ipam show

kubectl scale -n yaobank --replicas 1 deployments/customer
```

Enable encryption on cluster globally:

```
calicoctl patch felixconfiguration default --type='merge' -p '{"spec":{"wireguardEnabled":true}}'
```

Create our new nodegroup using Ubuntu 20.04:

```
eksctl create nodegroup --name calicoubuntu-ng --cluster calicocni --node-type t3.medium --node-ami-family Ubuntu2004 --max-pods-per-node 100 --ssh-access --version 1.22
```

View nodes:

```
eksctl get nodegroup --cluster calicocni

kubectl get nodes -o wide

ssh ubuntu@54.194.122.95 lsmod | grep wireguard
ssh ubuntu@54.194.122.95 sudo dmesg | grep wireguard
```

Cleanup:

```
eksctl delete cluster --name calicocni
```

### Self-Managed Installation Options

* Kubernetes Operations [`kOps`](https://kops.sigs.k8s.io/networking/calico/):
  * like `kubectl` clusters
  * build production grade clusters
  * HA clusters
  * provision necessary cloud infrastructure
  * acccess to all calico's features
  * AWS is officially supported
  * idempotent
  * less cloud independent that kubespray
* Other:
  * `kubeadm`:
    * minimum vialbe k8s cluster
    * more complex that kOps
  * [`kubespray`](https://www.tigera.io/blog/using-calico-with-kubespray/):
    * install k8s using `Ansible`
    * more complex that kOps
  * `k3s`:
    * lightweight k8s distribution (<100MB)
    * default to a `SQLite` instead of `etcd` datastore (less scalable)
  * `kind`
  * `minikube`
  * `microk8s`

### kOps overview

* [install kubectl and AWS CLI, then kOps](https://kops.sigs.k8s.io/getting_started/install/)
* [consider IAM, DNS and cluster state storage configuration in AWS](https://kops.sigs.k8s.io/getting_started/aws/)
* [for test cluster Gossip DNS can be used](https://kops.sigs.k8s.io/gossip/), for production Route53 can be used
* S3 is object storage, which can be used for cluster state storage in dedicated bucket (turn on versioning for rollback)
* [`--networking calico` while cluster creating](https://kops.sigs.k8s.io/networking/calico/):
  * `encapsulationMode`
  * `crossSubnet`
  * `mtu`
  * `bpfEnabled`
  * `wireguardEnabled`
