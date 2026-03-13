# Production-Grade Jenkins CI/CD on Amazon EKS
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
    minSize: 1
    maxSize: 5
    desiredCapacity: 1
    privateNetworking: true
    volumeSize: 50
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
### Jenkins helm values.yaml
Jenkins admin secret
```bash
kubectl create secret generic jenkins-admin-secret \
  --namespace jenkins \
  --from-literal=jenkins-admin-user=admin \
  --from-literal=jenkins-admin-password=<strong-password>
```
jenkins-values.yaml
```bash
controller:
  image: jenkins/jenkins
  tag: 2.452.2-lts-jdk-17
  resources:
    requests:
      cpu: '1'
      memory: 2Gi
    limits:
      cpu: '2'
      memory: 4Gi

nodeSelector:
  role: static
tolerations:
  - key: role
    value: static
    effect: NoSchedule

serviceAccountName: jenkins # IRSA for jenkins

# Admin credentials stores in k8s secrets
admin:
  existingSecret: jenkins-admin-secret
  userKey: jenkins-admin-user
  passwordKey: jenkins-admin-password

# JVM and java options (Disables the setup wizard and sets JVM heap correctly.)
javaOpts: '-Xms1024m -Xmx2048m -Djenkins.install.runSetupWizard=false'

# Pre-intall plugins
installPlugins:
  - kubernetes
  - workflow-aggregator
  - git
  - configuration-as-code
  - blueocean
  - sonar
  - aws-credentials
  - kubernetes-credentials
  - pipeline-stage-view
  - build-timeout
# Ingress
ingress:
  enabled: true
  ingressClassName: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: 'true'
  hostname: jenkins.gauravgirase.co.in
  tls:
    - secretName: jenkins-tls
      hosts:
        - jenkins.gauravgirase.co.in

# persistence storage
persistence:
  enabled: true
  storageClass: gp3
  size: 20Gi

agent:
  resources:
    requests:
      cpu: '500m'
      memory: 512Mi
    limits:
      cpu: '2'
      memory: 4Gi
  nodeSelector:
    role: jenkins-agent
  tolerations:
    - key: role
      value: jenkins-agent
      effect: NoSchedule
  podRetention: Never
  idleMinutes: 0
```
```bash
helm repo add jenkins https://charts.jenkins.io

helm upgrade --install jenkins jenkins/jenkins \
--namespace jenkins \
-f jenkins-values.yaml
```

## Custom Jenkins Agent pod template
Define specialized pod templates for different build types using jenkins
configuration as code (JCasC)
jcasc/jenkins-casc-configmap.yaml
```bash
jenkins:
  cloud:
    - kubernetes:
        name: kubernetes
        serverUrl: https://kubernetes.default.svc
        namespace: jenkins
        podTemplates:
          - name: maven-agent
            label: maven
            nodeSelector:
              role: jenkins-agent
            tolerations:
              - key: role
                value: jenkins-agent
                effect: NoSchedule
            containers:
              - name: maven
                image: maven:3.9-eclipse-temurin-17
                command: ["sleep"]
                args: ["99d"]
                resource_request_cpu: '1'
                resource_request_memory: '1Gi'
                resource_limit_cpu: '4'
                resource_limit_memory: '4Gi'
            volumes:
              - persistenceVolumeClaim:
                  claimName: maven-cache-efs   # make sure this exists
                  mountPath: /root/.m2
```

## SonarQube - Code Quality
**Prerequisites**
sonarqube requires vm.max_map_count=262144 on the host. Add this via a daemonset or node user data


### Sonar-qube-values.yaml
```bash
sonarqube:
  nodeSelector:
    role: static
  tolerations:
    - key: role
      value: static
      effect: NoSchedule
  resources:
    requests:
      cpu: '500m'
      memory: 2Gi
    limits:
      cpu: '2'
      memory: 4Gi
  persistence:
    enabled: true
    storageClass: gp3
    size: 20Gi
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    hosts:
      - name: sonarqube.gauravgirase.co.in
        path: /
        pathType: Prefix
    tls:
      - secretName: sonarqube-tls
        hosts:
          - sonarqube.gauravgirase.co.in

postgresql:
  persistence:
    enabled: true
    storageClass: gp3
    size: 10Gi
  nodeSelector:
    role: static
  tolerations:
    - key: role
      value: static
      effect: NoSchedule
```
### install 
```bash
helm add repo sonarqube https://SonarSource.github.io/helm-chart-sonarqube
helm upgrade --install sonarqube sonarqube/sonarquber \
--namespace sonarqube -f sonarqube-values.yaml
```

## ArgoCD - GitOps CD
argocd-values.yaml
```bash
global:
  nodeSelector:
    role: static
  tolerations:
    - key: role
      value: static
      effect: NoSchedule

server:
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
      nginx.ingress.kubernetes.io/backend-protocol: GRPC
    hosts:
      - argocd.gauravgirase.co.in
    tls:
      - secretName: argocd-tls
        hosts:
          - argocd.gauravgirase.co.in
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
```
### install
```bash
helm repo add argo https://argoproj.github.io/argo-helm

helm upgrade --install argocd argo/argo-cd \
--namespace argocd -f argocd-values.yaml
```

### Get initial password
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64
```
## ArgoCD Application Defination
```bash
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  labels:
    environment: production
    team: platform
spec:
  project: default
  source:
    repoURL: https://github.com/GauravGirase/gitops.git
    targetRevision: main
    path: k8s/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - createNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```
## Obervability stack (Prometheus + Grafana + Loki)
### Install kube-prometheus-stack
```bash
helm add repo prometheus-community https://prometheus-community.github.io/helm-chart
```
monitoring-values.yaml
```bash
global:
  nodeSelector:
    role: static
  tolerations:
    - key: role
      value: static
      effect: NoSchedule

prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          volumeMode: Filesystem
          resources: 
            requests:
              storage: 20Gi

grafana:
  persistence:
    enabled: true
    storageClassName: gp3
    size: 10Gi
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    hosts:
      - grafana.gauravgirase.co.in
        path: /
        pathType: Prefix
    tls:
      - secretName: grafana-tls
        hosts: 
          - grafana.gauravgirase.co.in
  grafana.ini:
    server: 
      root_url: 'https://grafana.gauravgirase.co.in'
  replicaCount: 2

alertManager:
  alertManagerSpec:
    replicas: 2
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          resources:
            requests:
              storage: 5Gi
```
### install
```bash
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
--namespace monitoring -f monitoring-values.yaml
```
### For scrap target
check: monitoing/monitoring.yaml

## Backup & Disaster Recovery (Velero)
### Install velero
```bash
eksctl create iamserviceaccount \
--name velero \
--namespace velero \
--cluster prod-cicd-cluster \
--attach-policy-arn arn:aws:iam::<ACCOUNT-ID>:policy/VeleroPolicy \
--approve
```

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts

helm upgrade --install velero vmware-tanzu/velero \
  --namespace velero \
  --set configuration.backupStorageLocation[0].provider=aws \
  --set configuration.backupStorageLocation[0].bucket=eks-velero-backup \
  --set configuration.backupStorageLocation[0].config.region=ap-south-1 \
  --set configuration.volumeSnapshotLocation[0].provider=aws \
  --set configuration.volumeSnapshotLocation[0].config.region=ap-south-1 \
  --set serviceAccount.server.create=false \
  --set serviceAccount.server.name=velero \
  --set initContainers[0].name=velero-plugin-for-aws \
  --set initContainers[0].image=velero/velero-plugin-for-aws:v1.9.0 \
  --set initContainers[0].volumeMounts[0].mountPath=/target \
  --set initContainers[0].volumeMounts[0].name=plugins
```
## Backup schedules
### Daily fill cluster backup (2am UTC)
```bash
velero schedule create daily-full \
--schedule='0 2 * * *' \
--ttl 720h # 30 Days retention
```
### Hourly jenkins backup
```bash
velero schedule create hourly-jenkins \
--schedule='0 * * * *' \
--include-namespace jenkins \
--ttl 72h
```
### Restor procedure 
```bash
velero restor create --from-backup daily-full-12336776878
```
**Note:** Test restore procedure monthly.

## External Secret Operator
### Create an IAM Policy for ExternalSecrets
secret-policy.json
```bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": "arn:aws:secretsmanager:ap-south-1:<ACCOUNT_ID>:secret:prod/*"
        }
    ]
}
```
```bash
kubectl create namespace external-secrets
```
```bash
eksctl create iamserviceaccount \
  --name external-secrets \
  --namespace external-secrets \
  --cluster prod-cicd-cluster \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/ExternalSecretsPolicy \
  --approve
```

### Sync AWS secret manager into k8s secrets automatically.

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm upgrade --install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set serviceAccount.create=false \
  --set serviceAccount.name=external-secrets
```

### Deploy secrets using aws secret manager
```bash
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore # defines where and how to access secrets
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
      auth:
        jwt: 
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: jenkins-github-token
  namespace: jenkins
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: jenkins-github-token # the name of the Kubernetes secret that will be created
  data:
    - secretKey: token
      remoteRef:
        key: prod/jenkins/github-token # AWS Secrets Manager secret name/path
        property: token # the specific key inside that secret JSON.
```