# #######################################################
# ## Assume AWS CLI, HELM, KOPS and CERTBOT are installed
# ########################################################

# 0 - Setup Project
## install CertBot and Route53 Plugin - Mac
```bash
brew install certbot
$( brew --prefix certbot )/libexec/bin/pip install certbot-dns-route53
```

## Set Environment Varibles
```bash
export KOPS_STATE_STORE=s3://kubernetes-kops-aaron
export SUB_DOMAIN=k8s.pwned.io
export CLUSTER_ENV=test
export NAME=$CLUSTER_ENV.$SUB_DOMAIN
export CERTBOT_DIR=$PWD/.certbotdir
export ENVNAME=tstlab
export ENV_DOMAIN=$ENVNAME.$NAME
export CERT_EMAIL=ajones@axway.com
```
## Creating a project dir
```bash
mkdir kops-cluster
cd kops-cluster
```


# 1 - Setup AWS IAM for KOPS
## Setup Groups and Policies
```bash
aws iam create-group --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
```
## 2 - Create AWS User and CReate Access Key
```bash 
aws iam create-user --user-name kops
aws iam add-user-to-group --user-name kops --group-name kops
aws iam create-access-key --user-name kops
```
## 3  - Configure the aws client to use your new IAM user
```bash
aws configure           # Use your new access and secret key here
aws iam list-users      # you should see a list of all your IAM users here

export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id --profile kops)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key --profile kops)
export AWS_REGION=us-west-2
```

## 4 - create zone on route53
**I have my domain registrar at namescheap.com so i created a subdomain k8s.pwned.io and added
a NS record to the name servers listed from AWS**
```bash
ID=$(uuidgen) && aws route53 create-hosted-zone --name k8s.pwned.io --caller-reference $ID --profile kops| jq .DelegationSet.NameServers
```
# Create Kubernetes Cluster Using KOPS
## 1 - create ssh keys for kops command
```bash
ssh-keygen -f kubecluster
```

## 2 - Use KOPS to create K8S Cluster
```bash
kops create cluster --name=$NAME \
--cloud aws \
--cloud-labels="Project=AaronJonesTesting" \
--state=$KOPS_STATE_STORE --zones=us-east-1a \
--master-size=t2.medium --master-volume-size=10 \
--node-size=t2.medium --node-volume-size=10 \
--node-count=2 \
--networking=weave \
--ssh-public-key=$PWD/kubecluster.pub \
--yes
```

## 3 - Wait for Cluster to be ready
```bash
time until kops validate cluster; do sleep 15 ; done
```

## 4 - Create Certs for Istio Ingrees Gateway
```bash
certbot certonly -nq --dns-route53 --agree-tos --email ${CERT_EMAIL} -d ${ENV_DOMAIN} \
--config-dir ${CERTBOT_DIR}/config --work-dir ${CERTBOT_DIR}/work --logs-dir ${CERTBOT_DIR}/logs

export PRIV_KEY=${CERTBOT_DIR}/config/archive/${ENV_DOMAIN}/privkey1.pem
export CERT=${CERTBOT_DIR}/config/archive/${ENV_DOMAIN}/fullchain1.pem
kubectl create namespace istio-system
kubectl create namespace apic-control
kubectl create secret tls ${ENVNAME}-k8s-certs -n istio-system --cert ${CERT} --key ${PRIV_KEY} -o yaml
```

## 5 - go to API Central and Create Service Environment
1. Login to https://apicentral.axway.com/
2. Go to Services and click add Environment button
3. Update the values as shown in the table below
4. Save and download the env kit

Name | Value
---- | -----
Environment name|My k8s
Name|My k8s
Protocol|https
Host| ${ENV_DOMAIN}
Port|443
Usage|publicly

## 6 - create keys for connecting to Control plane
### SMA keys
```bash
echo -n "Axway123" > password && \
openssl genrsa -des3 -out private_key.pem -passout file:password 2048 && \
openssl rsa -pubout -in  private_key.pem -passin file:password -out public_key.der -outform der && base64 public_key.der > public_key && \
kubectl create --namespace apic-control secret generic sma-secrets --from-file=publicKey=public_key --from-file=privateKey=private_key.pem --from-file=password="password" -o yaml
```
### CSA keys
```bash
openssl genpkey -algorithm RSA -out private_key.pem -pkeyopt rsa_keygen_bits:2048 && \
openssl rsa -pubout -in private_key.pem -out public_key.der -outform der && base64 public_key.der > public_key && \
kubectl create --namespace apic-control secret generic csa-secrets --from-file=publicKey=public_key --from-file=privateKey=private_key.pem --from-literal=password="" -o yaml
```

## 7 - Helm Setup
```bash
helm reset # only need if you have helm setup
```

### create service account
```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: kube-system
EOF
```
### setup tiller and install repos -- make sure default AWS region is us-west-2 per the s3 buck
```shell
helm plugin install https://github.com/hypnoglow/helm-s3.git
helm init --service-account tiller && \
helm repo add axway-public https://d3az62zktgbxuf.cloudfront.net/charts && \
helm repo add axway s3://axway-charts-us-west-2/public/charts && \
helm repo up
```


## 8 - install istio
```shell
helm upgrade --install --namespace istio-system istio  axway/istio -f ./istioOverride.yaml
```

## 9 - install api central agents
```shell
helm upgrade --install apic-hybrid axway/apicentral-hybrid --namespace apic-control \
-f ./hybridOverride.yaml --set observer.enabled=true --set observer.filebeat.sslVerification=none
```



## 10 - fix observer data
```shell
kubectl delete pod -n apic-control  -l name=fluent-bit
kubectl delete pod -n istio-system  -l app=telemetry
```

## 11 - deploy petstore
```shell
kubectl run -n apic-demo petstore --env SWAGGER_BASE_PATH=/v1 --image swaggerapi/petstore --expose=true --port 8080
kubectl label service petstore -n apic-demo apic-managed=true --overwrite
kubectl label service petstore -n apic-demo apic-managed-
```
## 12 - delete petstore
```shell
kubectl delete services -n apic-demo petstore
kubectl delete deployment -n apic-demo petstore
```
