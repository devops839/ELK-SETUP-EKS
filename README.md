# ELK-Cluster-Setup
![image](https://github.com/user-attachments/assets/b20b8656-777a-4ef0-b09d-06df37da037f)

## Prerequisites
Ensure the following tools are installed before proceeding:
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [EKSCTL](https://eksctl.io/introduction/installation/)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

## Install Required Tools
```sh
sudo apt update
sudo apt install -y unzip curl

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

# Install eksctl
curl -sSLO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz"
tar -xzf eksctl_Linux_amd64.tar.gz
sudo mv eksctl /usr/local/bin
eksctl version
```

## Configure AWS CLI
```sh
aws configure
```

## Create EKS Cluster
```sh
cd eks
eksctl create cluster -f eks-cluster.yaml
```

## Deploy Elasticsearch and Kibana using ECK
```sh
kubectl apply -f https://download.elastic.co/downloads/eck/2.16.1/operator.yaml
kubectl create -f https://download.elastic.co/downloads/eck/2.16.1/crds.yaml
```

## Deploy Elasticsearch
```sh
kubectl apply -f elastic.yaml
```

## Deploy Kibana
```sh
kubectl apply -f kibana.yaml
```

## Deploy Nginx Ingress Controller
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/aws/deploy.yaml
kubectl get pods -n ingress-nginx
kubectl get service -n ingress-nginx
```

## Install Cert-Manager
Cert-Manager automates the issuance and renewal of SSL/TLS certificates.
```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl get pods -n cert-manager
```

## Create a ClusterIssuer for Let’s Encrypt
Create a file named `cluster-issuer.yaml`:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: your-email@example.com  # Replace with your email
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```
Apply the ClusterIssuer:
```sh
kubectl apply -f cluster-issuer.yaml
```

## Configure the Kibana Ingress with Let's Encrypt
Create a file named `kibana-ingress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
    - host: kibana.pkdemo.org  # Replace with your actual domain
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prod-kb-svc
                port:
                  number: 5601
  tls:
    - hosts:
        - kibana.pkdemo.org
      secretName: kibana-tls-cert  # Cert-Manager will create this
```
Apply the Ingress:
```sh
kubectl apply -f kibana-ingress.yaml
```

## Validate the Certificate
```sh
kubectl get certificate
```
Expected output:
```
NAME              READY   SECRET               AGE
kibana-tls-cert   True    kibana-tls-cert      5m
```
If `READY = True`, your certificate is active.

## Test HTTPS Access
Now, open your Kibana UI:
🔗 `https://kibana.pkdemo.org`

### Login Credentials
- **Username**: `elastic`
- **Password**: Retrieve using the following command:
```sh
kubectl get secret prod-es-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo
```

## Delete EKS Cluster and Associated Volumes
To delete the EKS cluster and associated resources, run:
```sh
eksctl delete cluster --name=elk-cluster --region=us-east-1
```
This will remove all resources provisioned by the cluster, including worker nodes and networking components.

## Example Hosted URL
🔗 `https://kibana.pkdemo.org/`

