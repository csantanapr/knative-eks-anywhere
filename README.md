# Knative on EKS Anywhere

EKS Anywhere docs https://anywhere.eks.amazonaws.com

- Create Ubuntu
    ```bash
    multipass launch --name eks --cpus 4 --mem 16GB --disk 30G
    multipass exec eks -- lsb_release -a
    ```
- ssh into instance:
    ```bash
    multipass exec eks -- bash
    ```
- Install homebrew https://brew.sh/
- Install docker https://docs.docker.com/engine/install/ubuntu/
- Setup docker for user `ubuntu`
    ```bash
    sudo usermod -aG docker $USER
    newgrp docker
    ```

- Install EKS Anywhere https://anywhere.eks.amazonaws.com/docs/getting-started/install or use GitOps mode https://anywhere.eks.amazonaws.com/docs/tasks/cluster/cluster-flux/

Example:

<details><summary>dev-cluster.yaml</summary>

```yaml
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: Cluster
metadata:
  name: dev-cluster
spec:
  gitOpsRef:
    kind: GitOpsConfig
    name: my-dev-cluster
  clusterNetwork:
    cni: cilium
    pods:
      cidrBlocks:
      - 192.168.0.0/16
    services:
      cidrBlocks:
      - 10.96.0.0/12
  controlPlaneConfiguration:
    count: 1
  datacenterRef:
    kind: DockerDatacenterConfig
    name: dev-cluster
  externalEtcdConfiguration:
    count: 1
  kubernetesVersion: "1.21"
  managementCluster:
    name: dev-cluster
  workerNodeGroupConfigurations:
  - count: 1

---
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: DockerDatacenterConfig
metadata:
  name: dev-cluster
spec: {}

---
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: GitOpsConfig
metadata:
  name: my-dev-cluster
spec:
  flux:
    github:
      personal: true
      repository: eks-anywhere-clusters
      owner: csantanapr
```
</details>


Create cluster:
```bash
export EKSA_GITHUB_TOKEN=ghp_MyValidPersonalAccessTokenWithRepoPermissions
eksctl anywhere create cluster -f dev-cluster.yaml
```


<details><summary>Expected Output</summary>
```bash
Checking Github Access Token permissions
✅ Github personal access token has the required repo permissions
Performing setup and validations
Warning: The docker infrastructure provider is meant for local development and testing only
✅ Docker Provider setup is valid
✅ Flux path
✅ Create preflight validations pass
Creating new bootstrap cluster
Installing cluster-api providers on bootstrap cluster
Provider specific setup
Creating new workload cluster
Installing networking on workload cluster
Installing storage class on workload cluster
Installing cluster-api providers on workload cluster
Moving cluster management from bootstrap to workload cluster
Installing EKS-A custom components (CRD and controller) on workload cluster
Creating EKS-A CRDs instances on workload cluster
Installing AddonManager and GitOps Toolkit on workload cluster
Adding cluster configuration files to Git
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Compressing objects: 100% (4/4), done.
Total 5 (delta 0), reused 0 (delta 0), pack-reused 0
Finalized commit and committed to local repository      {"hash": "a02eb3973608aab0e40e6a4ed1e8ccb45dd4005e"}
Writing cluster config file
Deleting bootstrap cluster
🎉 Cluster created!
```

</details>

Use KUBECONFIG file to verify
```bash
export KUBECONFIG=${PWD}/dev-cluster/dev-cluster-eks-a-cluster.kubeconfig
kubectl get ns
```

- Configure LoadBalancer Kube-vip using this range and instructions https://anywhere.eks.amazonaws.com/docs/tasks/workload/loadbalance/kubevip/arp/
```bash
IP_START=172.18.0.100  # Use the starting IP in your range
IP_END=172.18.0.200  # Use the ending IP in your range
kubectl create configmap --namespace kube-system kubevip --from-literal range-global=${IP_START}-${IP_END}
```

- Install Knative
```bash
curl -sL https://raw.githubusercontent.com/csantanapr/knative-kind/master/02-serving.sh | bash
curl -sL https://raw.githubusercontent.com/csantanapr/knative-kind/master/02-kourier.sh | bash
kubectl apply -f https://storage.googleapis.com/knative-nightly/serving/latest/serving-default-domain.yaml
```

- Create a Knative Service
```bash
kn service create hello-eks --port 80 --image public.ecr.aws/aws-containers/hello-eks-anywhere:latest
```

- Try it out
```bash
SERVICE_URL=$(kubectl get ksvc hello-eks -o jsonpath='{.status.url}')
echo "The SERVICE_ULR is $SERVICE_URL"
curl $SERVICE_URL
```