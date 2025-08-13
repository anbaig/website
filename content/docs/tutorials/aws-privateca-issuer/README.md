---
title: Deploy cert-manager with AWS Private CA Issuer on Amazon EKS
description: |
    Learn how to deploy cert-manager with the AWS Private CA Issuer on Amazon Elastic Kubernetes Service (EKS)
    and configure it to issue certificates from AWS Private Certificate Authority for production workloads.
---

*Last Verified: 15 August 2025*

In this tutorial you will learn how to deploy and configure cert-manager with the AWS Private CA Issuer on Amazon Elastic Kubernetes Service (EKS). You will learn how to set up AWS Private Certificate Authority, configure the issuer, and issue certificates for your applications using a private PKI infrastructure.

The AWS Private CA Issuer is a cert-manager external issuer that integrates with AWS Private Certificate Authority to provide private certificates for your Kubernetes workloads. This is ideal for organizations that need to maintain their own private PKI infrastructure for internal services, microservices communication, or compliance requirements.

## Prerequisites

Before starting this tutorial, ensure you have:

- An AWS account with appropriate permissions
- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) installed and configured
- [kubectl v1.13+](https://kubernetes.io/docs/tasks/tools/install-kubectl/) installed
- [eksctl](https://eksctl.io/installation/) installed
- [Helm v3](https://helm.sh/docs/intro/install/) installed

## Configure the AWS CLI

Set up the AWS CLI for this tutorial:

```bash
aws configure
```

Set the default output format and region:

```bash
export AWS_DEFAULT_OUTPUT=json
export AWS_DEFAULT_REGION=us-west-2   # ❗ Replace with your preferred AWS region
```

## Create an EKS Kubernetes cluster

Create a Kubernetes cluster using EKS. Pick a name for your cluster and save it in an environment variable:

```bash
export CLUSTER=aws-privateca-demo
```

Create the cluster:

```bash
eksctl create cluster \
  --name $CLUSTER \
  --nodegroup-name node-group-1 \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed \
  --region $AWS_DEFAULT_REGION
```

This will update your `kubectl` config file with the credentials for your new cluster.

Check that you can connect to the cluster:

```bash
kubectl get nodes -o wide
```

> ⏲ It will take 15-20 minutes to create the cluster.
>
> 💵 This command creates a 3-node cluster using cost-effective virtual machines suitable for this tutorial.
>
> ⚠️ This cluster configuration is for learning purposes and may need additional hardening for production use.

## Install cert-manager

You can install cert-manager using either Helm or as an EKS add-on.

### Option 1: Install using Helm (Recommended for flexibility)

Install cert-manager using Helm:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.15.3 \
  --set crds.enabled=true
```

### Option 2: Install using EKS add-on

Alternatively, you can install cert-manager as an EKS add-on:

```bash
aws eks create-addon \
  --cluster-name $CLUSTER \
  --addon-name cert-manager \
  --addon-version v1.15.3-eksbuild.1 \
  --region $AWS_DEFAULT_REGION
```

Wait for the add-on to be active:

```bash
aws eks describe-addon \
  --cluster-name $CLUSTER \
  --addon-name cert-manager \
  --region $AWS_DEFAULT_REGION \
  --query 'addon.status'
```

### Verify the installation

Wait for cert-manager to be ready:

```bash
kubectl wait --for=condition=ready pods --all -n cert-manager --timeout=120s
```

Verify the installation:

```bash
kubectl -n cert-manager get all
```

You should see three deployments running: cert-manager, cert-manager-cainjector, and cert-manager-webhook.

> 📖 Read about [other ways to install cert-manager](../../installation/README.md).

## Create AWS Private Certificate Authority

You can either create a new AWS Private CA or use an existing one. For this tutorial, we'll create a new one.

### Option 1: Create a new AWS Private CA

Create a Private CA using the AWS CLI:

```bash
# Create the CA
CA_ARN=$(aws acm-pca create-certificate-authority \
  --certificate-authority-configuration file://ca-config.json \
  --certificate-authority-type "ROOT" \
  --query CertificateAuthorityArn --output text)

echo "Created CA with ARN: $CA_ARN"
```

First, create the CA configuration file:

```bash
cat > ca-config.json << EOF
{
  "KeyAlgorithm": "RSA_2048",
  "SigningAlgorithm": "SHA256WITHRSA",
  "Subject": {
    "Country": "US",
    "Organization": "Example Corp",
    "OrganizationalUnit": "IT Department",
    "CommonName": "Example Corp Root CA"
  }
}
EOF
```

Create the CA:

```bash
CA_ARN=$(aws acm-pca create-certificate-authority \
  --certificate-authority-configuration file://ca-config.json \
  --certificate-authority-type "ROOT" \
  --query CertificateAuthorityArn --output text)

echo "Created CA with ARN: $CA_ARN"
```

Get the CSR and create a self-signed root certificate:

```bash
# Get the CSR
aws acm-pca get-certificate-authority-csr \
  --certificate-authority-arn $CA_ARN \
  --output text > ca.csr

# Issue the root certificate
CERT_ARN=$(aws acm-pca issue-certificate \
  --certificate-authority-arn $CA_ARN \
  --csr fileb://ca.csr \
  --signing-algorithm "SHA256WITHRSA" \
  --template-arn "arn:aws:acm-pca:::template/RootCACertificate/V1" \
  --validity Value=10,Type="YEARS" \
  --query CertificateArn --output text)

# Wait for certificate to be issued
sleep 10

# Get the certificate
aws acm-pca get-certificate \
  --certificate-authority-arn $CA_ARN \
  --certificate-arn $CERT_ARN \
  --query Certificate --output text > ca.crt

# Import the certificate
aws acm-pca import-certificate-authority-certificate \
  --certificate-authority-arn $CA_ARN \
  --certificate fileb://ca.crt

# Clean up temporary files
rm ca-config.json ca.csr ca.crt
```

### Option 2: Use an existing AWS Private CA

If you already have an AWS Private CA, set the ARN:

```bash
export CA_ARN="arn:aws:acm-pca:us-west-2:123456789012:certificate-authority/12345678-1234-1234-1234-123456789012"
```

## Configure IAM permissions

The AWS Private CA Issuer needs permissions to interact with AWS Private CA. We'll use IAM Roles for Service Accounts (IRSA) for secure authentication.

### Create an IAM OIDC provider for your cluster

```bash
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER --approve
```

### Create an IAM policy

Create an IAM policy with the minimum required permissions:

```bash
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)

aws iam create-policy \
  --policy-name AWSPrivateCAIssuerPolicy \
  --description "Policy for AWS Private CA Issuer to manage certificates" \
  --policy-document file:///dev/stdin <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "acm-pca:DescribeCertificateAuthority",
        "acm-pca:GetCertificate",
        "acm-pca:IssueCertificate"
      ],
      "Resource": "$CA_ARN"
    }
  ]
}
EOF
```

### Create an IAM role and associate it with a Kubernetes service account

```bash
eksctl create iamserviceaccount \
  --name aws-privateca-issuer \
  --namespace aws-privateca-issuer \
  --cluster $CLUSTER \
  --role-name AWSPrivateCAIssuerRole \
  --attach-policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSPrivateCAIssuerPolicy \
  --approve
```

## Install AWS Private CA Issuer

You can install the AWS Private CA Issuer using either Helm or as an EKS add-on.

### Option 1: Install using Helm

Install the AWS Private CA Issuer using Helm:

```bash
helm repo add awspca https://cert-manager.github.io/aws-privateca-issuer
helm repo update

helm install aws-privateca-issuer awspca/aws-privateca-issuer \
  --namespace aws-privateca-issuer \
  --create-namespace \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-privateca-issuer
```

### Option 2: Install using EKS add-on

Alternatively, you can install the AWS Private CA Issuer as an EKS add-on:

```bash
aws eks create-addon \
  --cluster-name $CLUSTER \
  --addon-name aws-privateca-issuer \
  --region $AWS_DEFAULT_REGION \
  --service-account-role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/AWSPrivateCAIssuerRole
```

Wait for the add-on to be active:

```bash
aws eks describe-addon \
  --cluster-name $CLUSTER \
  --addon-name aws-privateca-issuer \
  --region $AWS_DEFAULT_REGION \
  --query 'addon.status'
```

### Verify the installation

Wait for the issuer to be ready:

```bash
kubectl wait --for=condition=ready pods --all -n aws-privateca-issuer --timeout=120s
```

Verify the installation:

```bash
kubectl -n aws-privateca-issuer get all
```

## Configure the AWS Private CA ClusterIssuer

Create a ClusterIssuer that references your AWS Private CA:

```yaml
apiVersion: awspca.cert-manager.io/v1beta1
kind: AWSPCAClusterIssuer
metadata:
  name: aws-privateca-cluster-issuer
spec:
  arn: ${CA_ARN}
  region: ${AWS_DEFAULT_REGION}
```

Apply the ClusterIssuer configuration:

```bash
envsubst < cluster-issuer.yaml | kubectl apply -f -
```

Check the status of the ClusterIssuer:

```bash
kubectl describe awspcaclusterissuer aws-privateca-cluster-issuer
```

You should see a status indicating that the issuer is ready:

```
Status:
  Conditions:
    Last Transition Time:  2025-08-13T21:00:00Z
    Message:               AWS PCA Issuer is ready
    Reason:                Verified
    Status:                True
    Type:                  Ready
```

## Issue your first certificate

Now that everything is configured, let's issue a certificate using the AWS Private CA Issuer.

Create a Certificate resource:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-certificate
  namespace: default
spec:
  secretName: example-certificate-tls
  issuerRef:
    name: aws-privateca-cluster-issuer
    kind: AWSPCAClusterIssuer
    group: awspca.cert-manager.io
  commonName: example.internal
  dnsNames:
  - example.internal
  - api.example.internal
  duration: 2160h # 90 days
  renewBefore: 360h # 15 days
  usages:
  - digital signature
  - key encipherment
  - server auth
```

Apply the certificate:

```bash
kubectl apply -f certificate.yaml
```

Check the status of the certificate:

```bash
kubectl get certificate example-certificate
kubectl describe certificate example-certificate
```

You should see the certificate in a "Ready" state:

```
NAME                 READY   SECRET                    AGE
example-certificate  True    example-certificate-tls   30s
```

Inspect the issued certificate:

```bash
kubectl get secret example-certificate-tls -o yaml
```

You can also decode and examine the certificate:

```bash
kubectl get secret example-certificate-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout
```

## Deploy a sample application with TLS

Let's deploy a simple web server that uses the certificate we just issued:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example-app
  template:
    metadata:
      labels:
        app: example-app
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 443
        volumeMounts:
        - name: tls-certs
          mountPath: /etc/nginx/ssl
          readOnly: true
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
      volumes:
      - name: tls-certs
        secret:
          secretName: example-certificate-tls
      - name: nginx-config
        configMap:
          name: nginx-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: default
data:
  default.conf: |
    server {
        listen 443 ssl;
        server_name example.internal;
        
        ssl_certificate /etc/nginx/ssl/tls.crt;
        ssl_certificate_key /etc/nginx/ssl/tls.key;
        
        location / {
            return 200 'Hello from AWS Private CA secured app!\n';
            add_header Content-Type text/plain;
        }
    }
---
apiVersion: v1
kind: Service
metadata:
  name: example-app-service
  namespace: default
spec:
  selector:
    app: example-app
  ports:
  - port: 443
    targetPort: 443
    protocol: TCP
  type: ClusterIP
```

Apply the application:

```bash
kubectl apply -f example-app.yaml
```

Test the application:

```bash
# Port forward to test locally
kubectl port-forward service/example-app-service 8443:443 &

# Test the connection (use --insecure since it's a private CA)
curl -k https://localhost:8443

# Stop port forwarding
kill %1
```

## Certificate renewal

cert-manager will automatically renew certificates before they expire. You can configure the renewal behavior using the `renewBefore` field in the Certificate spec.

To test certificate renewal, you can manually trigger it:

```bash
# Annotate the certificate to trigger renewal
kubectl annotate certificate example-certificate cert-manager.io/force-renew=$(date +%s)

# Watch the renewal process
kubectl get certificate example-certificate -w
```

## Cleanup

After completing the tutorial, clean up the resources:

```bash
# Delete the certificate and application
kubectl delete certificate example-certificate
kubectl delete -f example-app.yaml

# Delete the ClusterIssuer
kubectl delete awspcaclusterissuer aws-privateca-cluster-issuer

# Uninstall the AWS Private CA Issuer
# If installed via Helm:
helm uninstall aws-privateca-issuer -n aws-privateca-issuer

# If installed via EKS add-on:
# aws eks delete-addon --cluster-name $CLUSTER --addon-name aws-privateca-issuer --region $AWS_DEFAULT_REGION

# Uninstall cert-manager
# If installed via Helm:
helm uninstall cert-manager -n cert-manager

# If installed via EKS add-on:
# aws eks delete-addon --cluster-name $CLUSTER --addon-name cert-manager --region $AWS_DEFAULT_REGION

# Delete the EKS cluster
eksctl delete cluster --name $CLUSTER

# Delete the AWS Private CA (if you created one for this tutorial)
aws acm-pca update-certificate-authority \
  --certificate-authority-arn $CA_ARN \
  --status DISABLED

# Wait 7 days before you can delete the CA, or use --permanent-deletion-time-in-days 7 for immediate deletion
aws acm-pca delete-certificate-authority \
  --certificate-authority-arn $CA_ARN \
  --permanent-deletion-time-in-days 7

# Delete IAM resources
aws iam detach-role-policy \
  --role-name AWSPrivateCAIssuerRole \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSPrivateCAIssuerPolicy

aws iam delete-role --role-name AWSPrivateCAIssuerRole
aws iam delete-policy --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSPrivateCAIssuerPolicy
```

## Next Steps

Now that you have cert-manager working with AWS Private CA Issuer, you can:

1. [Configure an Ingress or mTLS between microservices using private certificates from AWS Private CA](https://github.com/aws-samples/sample-encryption-in-transit-for-kubernetes)

> 📖 Read more about [cert-manager concepts](../../concepts/README.md) and [other cert-manager tutorials](../README.md).
>
> 📖 Learn about [AWS Private CA best practices](https://docs.aws.amazon.com/acm-pca/latest/userguide/PcaBestPractices.html).
