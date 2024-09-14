#  KUBERNETES ETCD BACKUP AND NAMESPACE SECURITY
###  As an infrastructure admin, I was assigned the task of safeguarding a Kubernetes cluster by backing up its etcd data, ensuring network security within a specific namespace, restricting user access to view access only, and upgrading the cluster’s master node to the latest version. This project emphasized the importance of data integrity and security within cloud-native environments.

### Step 1: Backup the ETCD Cluster Data

```etcdctl snapshot save /tmp/myback --endpoints=https://172.31.21.203:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key```

```etcdctl snapshot status /tmp/myback --endpoints=https://172.31.21.203:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key```

![etcd-backup](./images/etcd-backup.png)

### Step 2: Creating the Namespace and Configuring Network Policies: The next task was to isolate a specific environment for the project within Kubernetes. I created a namespace named cep-project2. Afterward, I applied a strict network policy to ensure that only the Pods within this namespace could communicate with each other, blocking all external access

`kubectl create namespace cep-project2`

`vi network-policy.yaml`

`kubectl apply -f network-policy.yaml`

```kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-same-namespace
  namespace: cep-project2
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}
```

![namespace-creation](./images/namespace-creation.png)

![net-policy](./images/network-policy.png)

### This allowed only intra-namespace communication and denied access from Pods outside the cep-project2 namespace.

### Step 3: Configuring Kubernetes Client for User4 (Worker Node 3): The next challenge was to configure access for user4, ensuring that this user had read-only permissions on Worker Node 3. I generated the necessary certificates for user4 and used kubectl commands to assign only view access to the cep-project2 namespace.

`openssl genrsa -out user4.key 2048`

`openssl req -new -key user4.key -out user4.csr -subj "/CN=user4`

![generating-csr](./images/generating-cert.png)

#### Then i Created a Certificate Signing Request

`vi user4-csr.yaml`

```apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: user4
spec:
  request: 
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
```

`kubectl apply -f user4-csr.yaml`

![creating-csr](./images/creating-csr.png)

`cat user4.csr | base64 | tr -d "\n"`

#### This command gets the base64 encoded value of the CSR file content, copy and paste it in the request of the user4-csr.yaml file.

####  Approve the CertificateSigningRequest 

`kubectl certificate approve user4`

![approve-cert](./images/approve-cert.png)

#### Get Certificate and Export the issued certificate from the CertificateSigningRequest.

`kubectl get csr/user4 -o yaml`

`kubectl get csr user4 -o jsonpath='{.status.certificate}'| base64 -d > user4.crt`

### Create the user4.kubeconfig File Use the signed certificate (user4.crt), key (user4.key), and Kubernetes CA (ca.crt) to generate user4.kubeconfig

```kubectl config set-cluster my-cluster --server=https://172.31.21.203:6443 --certificate-authority=/etc/kubernetes/pki/ca.crt --kubeconfig=/home/labsuser/user4.kubeconfig
kubectl config set-credentials user4 --client-key=user4.key --client-certificate=user4.crt --embed-certs=true --kubeconfig=/home/labsuser/user4.kubeconfig
kubectl config set-context user4-context --cluster=my-cluster --user=user4 --namespace=cep-project2 --kubeconfig=/home/labsuser/user4.kubeconfig
kubectl config use-context user4-context --kubeconfig=/home/labsuser/user4.kubeconfig
```
#### I encountered an issue here where i did not specify the full path to my user4.kubeconfig file where it kept asking for username.

![setting-up](./images/user4-koncif.png)

### Step 5: Grant user4 View Access Create a role and rolebindingwith view access in the cep-project2 namespace

`kubectl create role view-role --verb=get,list,watch --resource=pods,services --namespace=cep-project2`

`kubectl create rolebinding user4-view-binding --role=view-role --user=user4 --namespace=cep-project2`

#### Then i created a deployment in the namespace

`kubectl create deployment nginx --image=nginx --namespace=cep-project2`

`kubectl scale deployment nginx --replicas=3 --namespace=cep-project2`

![creating-deploy](./images/creating-deployment.png)

### Step 6: I Configured a Kubernetes client on worker node 2 in such a way that user4 should have only view access to cep-project2. 
#### I copied my use4.kubeconfig file on my master node to my workernode2 and validated if user4 truly have view access only.

`kubectl --kubeconfig=/home/labsuser/.kube/user4.kubeconfig auth can-i get pods --namespace=cep-project2`

`kubectl --kubeconfig=/home/labsuser/.kube/user4.kubeconfig auth can-i create pods --namespace=cep-project2`

`kubectl --kubeconfig=/home/labsuser/.kube/user4.kubeconfig get pods --namespace=cep-project2`


`kubectl --kubeconfig=/home/labsuser/.kube/user4.kubeconfig create pods --namespace=cep-project2`

![validation](./images/validating-worker2.png)


