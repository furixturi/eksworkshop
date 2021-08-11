
# What is this
Basically this
https://www.eksworkshop.com/

# Getting started - Local

Note: for using Cloud9, see https://www.eksworkshop.com/020_prerequisites/workspace/

## Prerequisites
### Install tools

- if needed, install / upgrade AWS CLI
    ```sh
    $ python3 -m pip install --upgrade awscli && hash -r
    ```
- if needed, install jq, envsubst(from gettext)
    ```sh
    $ brew install jq gettext
    ```

- if needed, switch aws CLI profile
    ```sh
    $ aws configure list-profiles
    $ export AWS_PROFILE=xiaolshe-ops
    ```
- if needed, install kubectl
    ```sh
    $ brew install kubectl
    ```
    For [kubectl auto completion in zsh](https://kubernetes.io/docs/tasks/tools/included/optional-kubectl-configs-zsh/):
    Add this line in `~/.zshrc`
    ```sh
    source <(kubectl completion zsh)
    ```
    Then source `~/.zshrc` to enable it
    ```sh
    $ source ~/.zshrc
    ```
- if needed, [install eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html) 
    - Install Weaveworks Homebrew tap
    ```sh
    $ brew tap weaveworks/tap
    ```
    - Install or upgrade `eksctl`
    ```sh
    $ brew install weaveworks/tap/eksctl
    ```
    - for [auto complete in zsh](https://eksctl.io/introduction/)
      - run
        ```sh
        $ mkdir -p ~/.zsh/completion/
        $ eksctl completion zsh > ~/.zsh/completion/_eksctl
        ```
    - add this line to `~/.zshrc`
        ```
        fpath=($fpath ~/.zsh/completion)
        ```
    - source the `~/.zshrc`


### Create a KMS key
```sh
$ aws kms create-alias --alias-name alias/eksworkshop --target-key-id $(aws kms create-key --query KeyMetadata.Arn --output text)
```
Export it as an ENV
```sh
$ export MASTER_ARN=$(aws kms describe-key --key-id alias/eksworkshop --query KeyMetadata.Arn --output text)
```

### set ENVs
- ACCOUNT_ID
- AWS_REGION
- AZS
  ```sh
  $ export AZS=($(aws ec2 describe-availability-zones --query 'AvailabilityZones[].ZoneName' --output text --region $AWS_REGION))
  $ echo ${AZS[@]}
  ap-northeast-1a ap-northeast-1c ap-northeast-1d
  ```

### clone workshop repos
```sh
$ git clone https://github.com/aws-containers/ecsdemo-frontend.git
$ git clone https://github.com/brentley/ecsdemo-nodejs.git
$ git clone https://github.com/brentley/ecsdemo-crystal.git
```

## Launch
### Generate the cluster yaml
Note: 
- pay attention if the ENVs `$AWS_REGION` and `$AZS` are properly set
- bash array index starts from 1
```yaml
$ cat << EOF > eksworkshop.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkshop-eksctl
  region: ${AWS_REGION}
  version: "1.19"

availabilityZones: ["${AZS[1]}", "${AZS[2]}", "${AZS[3]}"]

managedNodeGroups:
- name: nodegroup
  desiredCapacity: 3
  instanceType: t3.small
  ssh:
    enableSsm: true

# To enable all of the control plane logs, uncomment below:
cloudWatch:
 clusterLogging:
   enableTypes: ["*"]

secretsEncryption:
  keyARN: ${MASTER_ARN}
EOF
```
### Create an EKS cluster
```sh
$ eksctl create cluster -f eksworkshop.yaml
```
Verify the nodes
```sh
$ kubectl get nodes
NAME                                                STATUS   ROLES    AGE   VERSION
ip-192-168-1-175.ap-northeast-1.compute.internal    Ready    <none>   25m   v1.19.6-eks-49a6c0
ip-192-168-33-172.ap-northeast-1.compute.internal   Ready    <none>   26m   v1.19.6-eks-49a6c0
ip-192-168-79-108.ap-northeast-1.compute.internal   Ready    <none>   25m   v1.19.6-eks-49a6c0
```

### Deploy a k8s dashboard to view the cluster
- Deploy a k8s native dashboard
    ```sh
    $ export DASHBOARD_VERSION="v2.0.0"
    $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/${DASHBOARD_VERSION}/aio/deploy/recommended.yaml

    namespace/kubernetes-dashboard created
    serviceaccount/kubernetes-dashboard created
    service/kubernetes-dashboard created
    secret/kubernetes-dashboard-certs created
    secret/kubernetes-dashboard-csrf created
    secret/kubernetes-dashboard-key-holder created
    configmap/kubernetes-dashboard-settings created
    role.rbac.authorization.k8s.io/kubernetes-dashboard created
    clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
    rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
    clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
    deployment.apps/kubernetes-dashboard created
    service/dashboard-metrics-scraper created
    deployment.apps/dashboard-metrics-scraper created
    ```

- Create a proxy to access the dashboard from local
  ```sh
  $ kubectl proxy --port=8080 --address=0.0.0.0 --disable-filter=true &
  ```
  Now if you open `http://localhost:8080` in browser, you see the k8s backend paths
- access the dashboard by appending the dashboard url 
    ```sh
    http://localhost:8000/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
    ```
    This will open the dashboard login page 
- get the token for dashboard login
    ```sh
    $ aws eks get-token --cluster-name eksworkshop-eksctl | jq -r '.status.token'
    ```
    Enter the result in the dashboard login page to access the dashboard

- to kill the proxy running on background
  ```sh
  # get it to foreground
  $ fg
  # ctrl+C
  ```
- more thorough clean up
    ```sh
    # kill proxy
    $ pkill -f 'kubectl proxy --port=8080'

    # delete dashboard
    $ kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/${DASHBOARD_VERSION}/aio/deploy/recommended.yaml

    $ unset DASHBOARD_VERSION
    ```