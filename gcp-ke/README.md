# Setup guide for ELK stack on GCP KE (Kubernets Engine)
Below guide is divided in to two parts being Kubernetes cluster creation and ELK stack deployment.

### Part I: Configuring and creating the Kubernetes cluster
Setting up of cluster is done via console. Please follow the [quickstart guide](https://cloud.google.com/kubernetes-engine/docs/quickstart) documentation.

### Part II: Deploying ELK stack
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
