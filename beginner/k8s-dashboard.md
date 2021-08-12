# Deploy a k8s dashboard to view the cluster
## Deploy the k8s native dashboard
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

## Create a proxy to access the dashboard from local
  ```sh
  $ kubectl proxy --port=8080 --address=0.0.0.0 --disable-filter=true &
  ```
  Now if you open `http://localhost:8080` in browser, you see the k8s backend paths
## access the dashboard by appending the dashboard url 
  ```sh
  http://localhost:8000/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
  ```
  This will open the dashboard login page 
### get the token for dashboard login
  ```sh
  $ aws eks get-token --cluster-name eksworkshop-eksctl | jq  -r '.status.token'
  ```
Enter the result in the dashboard login page to access the dashboard

## to kill the proxy running on background
  ```sh
  # get it to foreground
  $ fg
  # ctrl+C
  ```
## more thorough clean up
  ```sh
  # kill proxy
  $ pkill -f 'kubectl proxy --port=8080'

  # delete dashboard
  $ kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/${DASHBOARD_VERSION}/aio/deploy/recommended.yaml

  $ unset DASHBOARD_VERSION
  ```