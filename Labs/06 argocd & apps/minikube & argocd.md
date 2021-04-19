## <font color='red'> 1.1 ArgoCD </font>
Lab demonstrates a possible GitOps workflow using Argo CD and Tekton. We are using Argo CD to setup our Kubernetes clusters dev and prod (in the following we will only use the dev cluster) and Tekton to build and update our example application.

In this lab we're going to:
* install k8s-argocd cluster
* configure to pull apps from GitHub - GitOps
* sync apps on k8s-argocd cluster
* metrics in Grafana & Prometheus

---

#### <font color='red'>IMPORTANT:</font> 
<strong>Please ensure you start with a clean environment. 
If you have previously run minikube, you will need to delete the existing instance.</strong>

to stop  minikube:
```
minikube stop
```
to delete  minikube:
```
minikube delete
```

---

Pre-requisties:
* Kustomize lets you lets you create an entire Kubernetes application out of individual pieces — without touching the YAML configuration filesfor the individual components.  For example, you can combine pieces from different sources, keep your customizations — or kustomizations, as the case may be — in source control, and create overlays for specific situations. And it is part of Kubernetes 1.14 or later. Kustomize enables you to do that by creating a file that ties everything together, or optionally includes “overrides” for individual parameters.

ensure Snap is installed:
```
sudo yum install epel-release
sudo yum install snapd
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap
```
install Kustomize:
```
sudo snap install kustomize
```

* GitOps is a way to do Kubernetes cluster management and application delivery.  It works by using Git as a single source of truth for declarative infrastructure and applications. With GitOps, the use of software agents can alert on any divergence between Git with what's running in a cluster, and if there's a difference, Kubernetes reconcilers automatically update or rollback the cluster depending on the case. With Git at the center of your delivery pipelines, developers use familiar tools to make pull requests to accelerate and simplify both application deployments and operations tasks to Kubernetes.

obviously you will require a GitHub account if you want to try yourself.

* Docker Hub


---

#### <font color='red'> 1.1.1 ArgoCD - Dev K8s Cluster </font>
The next step is to create k8s-dev Kubernetes cluster: 
* k8s-dev - install ArgoCD

start k8s-dev cluster:
```
minikube start -p k8s-dev
```
enable ingress:
```
minikube addons enable ingress -p k8s-dev
```
verify ingress:
```
ksysgpo
```
confirm that your k8s-dev context is set correctly:
```
kubectl config use-context k8s-dev
```

---

#### <font color='red'> 1.1.2 Install ArgoCD + Apps </font>
ArgoCD is a declarative GitOps tool built to deploy applications to Kubernetes. While the continuous delivery (CD) space is seen by some as crowded these days, ArgoCD does bring some interesting capabilities to the table.

Unlike other tools, ArgoCD is lightweight and easy to configure. It is purpose-built to deploy applications to Kubernetes so it doesn’t have the UI overhead of many tools built to deploy to multiple locations.

In this lab we're going to:
* install ArgoCD
* deploys Prometheues Stack via kube-prometheus-stack helm chart
* install ArgoCD CLI


install ArgoCD:
```
kustomize build clusters/argocd/dev | k apply -f -
```
verify deployed ArgoCD:
```
kgpo -n argocd
```
deploy our manifests to the cluster using the app of apps pattern. 
create a new application, which manages all other applications (including ArgoCD):
```
k apply -f clusters/apps/dev.yaml
```
add our Ingresses to the /etc/hosts file:
```
sudo echo "`minikube ip -p k8s-dev` argocd-dev.fake grafana-dev.fake prometheus-dev.fake tekton-dev.fake server-dev.fake" | sudo tee -a /etc/hosts
```
Note: just execute once. check /etc/hosts file.

  > open in browser: http://argocd-dev.fake

user: admin

initial password is autogenerated:
```
k -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
Note: In the UI of Argo CD we can now see all deployed applications.

---

#### <font color='red'> 1.1.3 Install ArgoCD </font>
This example also deploys the Prometheus Stack via the [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) Helm chart. We are using the [Flux Helm Operator](https://docs.fluxcd.io/projects/helm-operator/en/stable/) instead of the Argo CD to deploy the Helm chart. When the Helm chart was successfully synced Prometheus is available at [prometheus-dev.fake](https://prometheus-dev.fake) and Grafana at [grafana-dev.fake](https://grafana-dev.fake).

log into Grafana:

  > in browser: http://grafana-dev.fake

user: admin  
password: admin

import dashboard for ArgoCD:

The dashboard can be found in the GitHub repository of the Argo CD project at [https://github.com/argoproj/argo-cd/blob/master/examples/dashboard.json](https://github.com/argoproj/argo-cd/blob/master/examples/dashboard.json).

---

#### <font color='red'>1.2.2 Install ArgoCD CLI </font>

go to the releases site on GitHub:

  > https://github.com/argoproj/argo-cd/releases

select the correct version (currently Tags:1.8.7):

rename the file and move to $PATH:
```
cd Downloads
mv argocd-linux-amd64 argocd 
sudo mv argocd /usr/local/bin
```
log into ArgoCD:
```
argocd login localhost:8080
```
or
  > http://localhost:8080

if you want to change the password:
```
argocd account update-password
```
tell ArgoCD about your deployment target:
```
argocd cluster add k8s-dev
```
Note: The above command installs a ServiceAccount (argocd-manager), into the kube-system namespace of that kubectl context, and binds the service account to an admin-level ClusterRole. Argo CD uses this service account token to perform its management tasks (i.e. deploy/monitoring).

---

#### <font color='red'>1.2.3 Build Demo Application </font>

---

#### <font color='red'>1.2.4 Add Demo Application to ArgoCD</font>

set up a couple of environment variables:
```
export ARGOCD_OPTS='--port-forward-namespace argocd'
```
Note: This variable will tell the argocd CLI client where our ArgoCD installation resides
set minikube dev ip:
```
export MINIKUBE_IP=https://$(minikube ip -p k8s-dev):8443
```
Note: This variable sets target cluster API URL.
create the application record:
```
argocd app create guestbook --repo https://github.com/jporeilly/ArgoCD-demo.git --path guestbook --dest-server $MINIKUBE_IP --dest-namespace default
```
verify status and configuration of your app:
```
argocd app list
```
Notice: the STATUS: OutOfSync and HEALTH: Missing. That’s because ArgoCD creates applications with manual triggers by default.  

“Sync” is the terminology ArgoCD uses to describe the application on your target cluster as being up to date with the sources ArgoCD is pulling from. You have set up ArgoCD to monitor the GitHub repository with the configuration files as well as the spring-petclinic container image in Docker Hub. Once the initial sync is completed, a change to either of these sources will cause the status in ArgoCD to change to OutOfSync.

more detailed view of your application configuration:
```
argocd app get guestbook
```
ready to sync your application to your target cluster:
```
argocd app sync guestbook
```
change kubectl contexts to your target cluster:
```
kubectl config use-context k8s-dev
```
port-forward to expose app on localhost:9090:
```
kubectl port-forward svc/guestbook -n default 9090:8080
```









