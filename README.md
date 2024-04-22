# EKS workshop demo basics

## Prerequisites

### AWS Cloud

- create AWS [account](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-creating.html)
- create IAM user in [Console](https://docs.aws.amazon.com/streams/latest/dev/setting-up.html#setting-up-iam)
- [configure](https://docs.aws.amazon.com/cli/latest/userguide/sso-configure-profile-token.html) and get IAM user temporary [credentials](https://docs.aws.amazon.com/singlesignon/latest/userguide/howtogetcredentials.html#how-to-get-temp-credentials)

### Local machine

- install [kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)
- install [eksctl](https://eksctl.io/installation/)
- install and setup [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- create EKS cluster via [eksctl](https://www.eksworkshop.com/docs/introduction/setup/your-account/using-eksctl)

```bash
# Create repo to store YAML template files
mkdir demo-eks && cd demo-eks

# Set AWS_PROFILE environment variable
export AWS_PROFILE=demo-eks

# Set EKS_CLUSTER_NAME environment variable
export EKS_CLUSTER_NAME=demo-eks

# Set EKS_CLUSTER_NAME environment variable
export AWS_REGION=eu-central-1

# Create the Cluster manifest:
cat <<EOF > ./cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
availabilityZones:
  - ${AWS_REGION}a
  - ${AWS_REGION}b
  - ${AWS_REGION}c
metadata:
  name: ${EKS_CLUSTER_NAME}
  region: ${AWS_REGION}
  version: "1.29"
  tags:
    karpenter.sh/discovery: ${EKS_CLUSTER_NAME}
    created-by: eks-workshop-v2
    env: ${EKS_CLUSTER_NAME}
iam:
  withOIDC: true
vpc:
  cidr: 10.1.0.0/16
  clusterEndpoints:
    privateAccess: true
    publicAccess: true
addons:
  - name: vpc-cni
    version: 1.18.0
    configurationValues: '{"env":{"ENABLE_PREFIX_DELEGATION":"true", "ENABLE_POD_ENI":"true", "POD_SECURITY_GROUP_ENFORCING_MODE":"standard"},"enableNetworkPolicy": "true"}'
    resolveConflicts: overwrite
managedNodeGroups:
  - name: ${EKS_CLUSTER_NAME}-ng
    desiredCapacity: 1
    minSize: 1
    maxSize: 3
    instanceType: t3a.medium
    privateNetworking: true
    releaseVersion: "1.29.0-20240129"
    updateConfig:
      maxUnavailablePercentage: 50
    labels:
      workshop-default: "yes"
EOF

# Create EKS Cluster via eksctl. Wait approx. 45 min
eksctl create cluster -f cluster.yaml

# Update EKS Cluster Kubeconfig
aws eks update-kubeconfig --name $EKS_CLUSTER_NAME

# Verify access to the Cluster
kubectl get node -o wide
kubectl get po -A
```

## Create Namespace

```bash
# Create the Namespace
kubectl create namespace demo

# Verify the Namespace creation:
kubectl get namespace | grep demo
```

## Create Pod

```bash
# Create the Pod:
kubectl -n demo run nginx --image=nginx --port=80 --restart=Never

# Verify the Pod creation:
kubectl -n demo get pod | grep nginx

# Check the Pod logs:
kubectl -n demo logs --follow nginx

# Delete the Pod:
kubectl -n demo delete pod nginx

# Verify the Pod deletion:
kubectl -n demo get pod

# Alternatively create the Pod manifest:
cat <<EOF > ./pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
EOF

# Create the Pod using kubectl apply:
kubectl -n demo apply -f ./pod.yaml

# Describe the Pod:
kubectl -n demo describe pod nginx
```

## Create Secret

- [K8s secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Managing k8s secrets](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-config-file/)

### Never Store Secrets In Plain Text! Following is for demonstation purposes only

```bash
# Convert the strings to base64:
echo -n 'admin' | base64
echo -n 't0p-Secret' | base64

# Create the Secret manifest:
cat <<EOF > ./secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: demo-auth-secret
type: Opaque
data:
  username: YWRtaW4= # put converted 'admin' data here
  password: dDBwLVNlY3JldA== # put converted 't0p-Secret' data here
EOF

# Alternatively create the basic-auth type Secret:
cat <<EOF > ./secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: demo-auth-secret
type: kubernetes.io/basic-auth
stringData:
  username: admin # required field for kubernetes.io/basic-auth
  password: t0p-Secret # required field for kubernetes.io/basic-auth
EOF

# Create the Secret using kubectl apply:
kubectl -n demo apply -f ./secret.yaml

# Verify the Secret creation:
kubectl -n demo get secret

# Verify the Secret data:
kubectl -n demo get secret demo-auth-secret -o jsonpath='{.data.username}' | base64 --decode
kubectl -n demo get secret demo-auth-secret -o jsonpath='{.data.password}' | base64 --decode
```

## Create Demployment

- [K8s deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

```bash
# Create the Demployment manifest:
cat <<EOF > ./demployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-nginx
  template:
    metadata:
      labels:
        app: demo-nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        env:
        - name: AUTH_USERNAME
          valueFrom:
            secretKeyRef:
              name: demo-auth-secret
              key: username
        - name: AUTH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: demo-auth-secret
              key: password
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
EOF

# Create the Demployment using kubectl apply:
kubectl -n demo apply -f ./demployment.yaml

# Verify the Demployment creation:
kubectl -n demo get demployment

# Describe the Demployment:
kubectl -n demo describe deployment demo-nginx-deployment
```

## Create Service

- [K8s service](https://kubernetes.io/docs/concepts/services-networking/service/)

```bash
# Create the Service manifest:
cat <<EOF > ./service.yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-nginx-service
spec:
  selector:
    app: demo-nginx
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
EOF

# Create the Service using kubectl apply:
kubectl -n demo apply -f ./service.yaml

# Expose the Service
kubectl -n demo port-forward service/demo-nginx-service 7080:8080

# Demo application is now available at http://localhost:7080/
curl -v http://localhost:7080/

# Alternatively expose the Service with public DNS name (creates LoadBalancer)
kubectl -n demo expose deployment demo-nginx-deployment --type LoadBalancer --name demo-nginx-lb

# Demo application is now available at public URL
curl -v "$(kubectl -n demo get service demo-nginx-lb -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
```

## Cleanup

```bash
# Delete all resources defined in YAML manifests
kubectl -n demo delete all --all

# Delete K8s cluster via eksctl
eksctl delete cluster --name $EKS_CLUSTER_NAME
```

## Getting Started

- [Getting started with Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html)
- [EKS workshop](https://www.eksworkshop.com/)
- [Key Kubernetes Concepts](https://www.tigera.io/blog/key-kubernetes-concepts/)
- [Introduction to Kubernetes](https://www.edx.org/learn/kubernetes/the-linux-foundation-introduction-to-kubernetes)
- [Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
- [CKAD excersices](https://github.com/dgkanatsios/CKAD-exercises)
- [EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)

## kubectl Quick Reference

- [Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

## Play with Kubernetes

- [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
- [Play with Kubernetes](https://labs.play-with-k8s.com/)
