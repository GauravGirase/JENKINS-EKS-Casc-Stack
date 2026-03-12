# Prodcution-Grade Jenkins CI/CD on Amazon EKS
**Covers:** EKS cluster | Jenkins controller + Agents | SonarQube | ArgoCD | Prometheus | Grafana | Cert-Manager | Ingress | Velero Backup | IRSA Security

## Install dependencies

## EKS cluster config
```bash
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: prod-cicd-cluster
  region: ap-south-1
  version: "1.34"
  tags:
    Environment: production
    Team: Platform

availabilityZones:
  - ap-south-1a
  - ap-south-1b

iam:
  withOIDC: true

managedNodeGroups:

  # system/infrastructure nodes
  - name: system-nodes
    instanceType: t3.small
    minSize: 2
    maxSize: 5
    desiredCapacity: 2
    privateNetworking: true
    volumeSize: 10
    volumeType: gp3
    labels:
      role: system
    taints:
      - key: role
        value: system
        effect: NoSchedule
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

  # static workload nodes
  - name: static-workload-nodes
    instanceType: t3.small
    minSize: 2
    maxSize: 4
    desiredCapacity: 2
    privateNetworking: true
    volumeSize: 20
    volumeType: gp3
    labels:
      role: static
    taints:
      - key: role
        value: static
        effect: NoSchedule

  # Jenkins agent nodes (spot)
  - name: jenkins-agent-nodes
    instanceTypes:
      - m5.large
      - c5.large
      - m5a.large
      - t2.medium
      - t3.medium
    spot: true
    minSize: 0
    maxSize: 10
    desiredCapacity: 0
    privateNetworking: true
    volumeSize: 20
    volumeType: gp3
    labels:
      role: jenkins-agent
    taints:
      - key: role
        value: jenkins-agent
        effect: NoSchedule
```
### Create EKS cluster 
```bash
eksctl create cluster -f cluster.yaml
```
### Update kube-config
```bash
aws eks update-kubeconfig --name prod-cicd-cluster --region ap-south-1
```
### Verify all nodes are ready
```bash
kubectl get nodes -o wide
```

### StorageClass Setup
create gp3 storageclass as the cluster default
```bash
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com  # dynamically create EBS volumes for pods.
parameters:
  type: gp3
  encrypted: "true"
# This ensures the volume is created in the same AZ as the pod. Without this, pods may fail with:
# volume node affinity conflict
volumeBindingMode: WaitForFirstConsumer  
allowVolumeExpansion: true # Allows resizing PVCs without deleting them.
```
**You can tune IOPS and throughput for gp3**
Useful for heavy workloads like: **SonarQube,Prometheus,Jenkins volume**
```bash
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
```
```bash
kubectl apply -f storageclass.yaml
```
## Cluster Autoscaler
Automatically adjusts the number of nodes based on pending pod demand. it scales jenkins agent nodes to zero when idle.
### Create IRSA role for cluster automscaler
```bash
eksctl create iamserviceaccount \
--name cluster-autoscaler \
--namespace kube-system \
--cluster prod-cicd-cluster \
--attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/ClusterAutoscalerPolicy \
--approve --override-existing-serviceaccounts
```
### Install via Helm
```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm upgrade --install cluster-autoscaler autoscaler/cluster-autoscaler \
--namespace kube-system \
--set autoDiscovery.clusterName=prod-cicd-cluster \
--set awsRegion=ap-south-1 \
--set rbac.serviceAccount.create=false \
--set rbac.serviceAccount.name=cluster-autoscaler \
--set extraArgs.scale-down-delay-after-add=5m \
--set extraArgs.scale-down-unneeded-time=3m
```
### Install NGINX Ingress controller
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
--namespace ingress-nginx --create-namespace \
--set controller.replicaCount=2 \
--set controller.nodeSelector.role=system \
--set controller.tolerations[0].key=role \
--set controller.tolerations[0].value=system \
--set controller.tolerations[0].effect=NoSchedule \
--set controller.service.type=LoadBalancer \
--set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"=nlb
```
This creates an AWS Network Load Balancer in AWS Asia Pacific (Mumbai) Region.
**Benefits:**
- ✔ static IP
- ✔ high performance
- ✔ layer 4 routing

### Install cert-manager (TLS Automation)
```bash
helm repo add jetstack https://charts.jetstack.io

helm upgrade --install cert-manager jetstack/cert-manager \
--namespace cert-manager --create-namespace \
--set installCRDs=true \
--set nodeSelector.role=system \
--set tolerations[0].key=role \
--set tolerations[0].value=system \
--set tolerations[0].effect=NoSchedule
```
### Create clusterIssuer for lets encrypt
```bash
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: gauravdevops1997@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - dns01:
          route53:
            region: ap-south-1
            hostedZoneID: ZXXXXXXXXXXXX   # Replace this
```
### How DNS-01 Works
```bash
Ingress created --> cert-manager requests certificate --> Route53 TXT record created --> Let's Encrypt verifies domain --> Certificate issued --> TLS secret created in Kubernetes
```
### ExternalDNS
It automatically creates Route 53 dns records for ingress resources

```bash
aws iam create-policy \
--policy-name ExternalDNSPolicy \
--policy-document file://external-dns-policy.json
```
```bash
eksctl create iamserviceaccount \
--cluster prod-cicd-cluster \
--namespace kube-system \
--name external-dns \
--attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/ExternalDNSPolicy \
--approve
```

```bash
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/

helm upgrade --install external-dns external-dns/external-dns \
--namespace kube-system \
--set provider=aws \
--set aws.zoneType=public \
--set domainFilters[0]=gauravgirase.co.in \
--set txtOwnerId=prod-cicd-cluster \
--set serviceAccount.create=false \
--set serviceAccount.name=external-dns
```
## Namespaces ,RBAC & Network Policies
Create all namespaces
```bash
kubectl create ns jenkins
kubectl create ns sonarqube
kubectl create ns argocd
kubectl create ns monitoring
kubectl create ns velero
```
## Network Policy Example (Jenkins namespace)
```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: jenkins-network-policy
  namespace: jenkins
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports: [{port: 8080, protocol: TCP}]
    - from:
      - podSelector: {}
  egress:
    - {} # allow all egress, Jenkins agents pulling Docker images, plugins, etc.
```
## Jenkins - Controller & k8s agents
Jenkins uses the k8s plugin to dynamically provision ephemeral build agent pods. Each pipeline job gets itw own pod, which is destroyed after the job completes. The controller runs as a statefulset with persistent storage
### Create Jenkins IRSA Role
**why ?**
Jenkins needs IAM access to ECR, s3, and Secret managers.
```bash
eksctl create iamserviceaccount \
--name jenkins \
--namespace jenkins \
--cluster prod-cicd-cluster \
--attach-policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess \
--attach-policy-arn arn:aws:iam::aws:policy/SecretManagerReadWrite \
--attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
--approve
```