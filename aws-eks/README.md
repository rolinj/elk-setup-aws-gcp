# Setup guide for ELK stack on AWS EKS via aws-cli
Below guide is divided in to two parts being EKS cluster creation and ELK stack deployment. Please also don't forget change the values for items enclosed with `<>` when copying code snippets.

### Pre-requisites:
- [Installing `helm`](https://helm.sh/docs/intro/install/) 
- [Installing `kubectl`](https://kubernetes.io/docs/tasks/tools/#kubectl)

### Part I: Configuring and creating the EKS cluster
1. Installing `eksctl` and `aws-cli`.
```
> /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
> brew tap weaveworks/tap
> brew install weaveworks/tap/eksctl
> aws configure (this is where the kubeconfig file gets updated)
> source <(kubectl completion zsh)
```
2. Creating local SSH keys and then importing to AWS.
```
> ssh-keygen -t rsa -C "<key_name_here>" -f ~/.ssh/<key_name_here>
> aws ec2 import-key-pair --key-name "<key_name_here>" --public-key-material fileb://~/.ssh/<key_name_here>.pub
```
The prefix `fileb://` is important so the file content will be considered as unencoded binary.

3. Creating the EKS cluster using the PUB key from previous step.
```
# eksctl create cluster --name <eks_cluster_name> --region us-east-2 --with-oidc --ssh-access --ssh-public-key ~/.ssh/<key_name_here>.pub --managed
```
4. Adding the `VPC CNI` Amazon EKS add-on.
```
> aws eks create-addon --cluster-name <eks_cluster_name> --addon-name vpc-cni
```
5. Adding the `CoreDNS` Amazon EKS add-on.
```
> aws eks create-addon --cluster-name <eks_cluster_name> --addon-name coredns
```
6. Adding the `kube-proxy` Amazon EKS add-on.
```
> aws eks create-addon --cluster-name <eks_cluster_name> --addon-name kube-proxy
```


### Part II: Deploying the ELK stack
1. Installing `ECK operator` using Helm.
```
> helm repo add elastic https://helm.elastic.co
> helm repo update
> helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
```
2. Installing `Elasticsearch`.
```
cat <<EOF | kubectl apply -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
spec:
  version: 7.13.1
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
EOF
```
3. Installing `Kibana`.
```
cat <<EOF | kubectl apply -f -
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: quickstart
spec:
  version: 7.13.1
  count: 1
  elasticsearchRef:
    name: quickstart
EOF
```
4. Exposing Kibana via Load Balancer.
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: kibana-tanaw-opswerks
spec:
  type: LoadBalancer
  selector:
    common.k8s.elastic.co/type: kibana
  ports:
    - name: "kibana-http"
      port: 5601
      targetPort: 5601
EOF
```
5. Retrieving the `External-IP` endpoint. 
```
> kubectl get svc kibana-tanaw-opswerks |  awk {'print $1" " $2 " " $4 " " $5'} | column -t
```
You may now access Kibana via `https://` and appending the port `:5601`. (Ex. `https://<your_external_endpoint>:5601`)

6. Retrieving the secrets password for elastic user to login.
```
> kubectl get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode
```

Default username is `elastic`.

![ELK Login Page](/aws-eks/elk_login_page.png)

