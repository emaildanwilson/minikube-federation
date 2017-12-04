# minikube-federation
Demo of Kubernetes Federation using Minikube &amp; CoreDNS

## 0. Prerequisites

- Minikube >= v0.23.0: https://github.com/kubernetes/minikube/releases/tag/v0.23.0

- Kubectl & kubefed  >= v1.8: https://kubernetes.io/docs/tasks/federation/set-up-cluster-federation-kubefed/#getting-kubefed

- Helm: https://github.com/kubernetes/helm#install

## 1. Clone this repo

The repo stores a few configuration files and the k8s yaml for a sample federated app

  ```console
  git clone https://github.com/emaildanwilson/minikube-federation
  cd ./minikube-federation
  ```

## 2. Create the minikube cluster

This cluster will store the federation api/controller, etcd v2 and coredns.

  ```console
  minikube start -p minikube
  ```

###### 2.1 deploy etcd v2 to the minikube cluster
etcd is used by a federation plugin to write dns records.

  ```console
  kubectl run etcd --image=quay.io/coreos/etcd:v2.3.7 --env="ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379" --env="ETCD_ADVERTISE_CLIENT_URLS=http://etcd.default:2379" --port=2379 --expose --context=minikube
  ```

###### 2.2 deploy coredns to the minikube cluster

We'll install and configure coredns using helm.
coredns then reads from etcd as its dnszone configuration.

  ```console
  helm init --kube-context minikube
  ```

Wait until `helm version` works.

  ```console
  Client: &version.Version{SemVer:"v2.6.1", GitCommit:"bbc1f71dc03afc5f00c6ac84b9308f8ecb4f39ac", GitTreeState:"clean"}
  Server: &version.Version{SemVer:"v2.6.1", GitCommit:"bbc1f71dc03afc5f00c6ac84b9308f8ecb4f39ac", GitTreeState:"clean"}
  ```

Inspect the `coredns-values.yaml` file to see how coredns is configured to obtain dns records from etcd.

  ```console
  cat ./coredns-values.yaml
  ```

Install coredns using helm.

  ```console
  helm install --namespace default --name coredns --kube-context minikube -f ./coredns-values.yaml stable/coredns
  ```

Verify that coredns is running

```console
kubectl get svc,pods -l app=coredns-coredns --context=minikube
```

## 3. Start additional minikube clusters

3.1 These clusters will be joined to federation and receive objects from the federation controllers.

  ```console
  minikube start -p us
  minikube start -p europe
  ```

3.2 label zones & regions on the nodes (cloud provider magic)

  ```
  kubectl label node us failure-domain.beta.kubernetes.io/zone=us1 failure-domain.beta.kubernetes.io/region=us --context=us
  kubectl label node europe failure-domain.beta.kubernetes.io/zone=europe1 failure-domain.beta.kubernetes.io/region=europe --context=europe
  ```

3.3 Add configmap objects used by federated ingress.
The uid is normally globally unique.

  ```console
  kubectl create configmap ingress-uid --from-literal=uid=us1 -n kube-system --context=us
  kubectl create configmap ingress-uid --from-literal=uid=europe1 -n kube-system --context=europe
  ```

# 4. Setup and Configure Federation

###### 4.1 Start the Federation API and Controller

Notice that the dns provider config is passed in from the local file coredns-provider.conf.

  ```console
  kubefed init myfed --host-cluster-context=minikube --dns-provider="coredns" --dns-zone-name="myzone." --api-server-service-type=NodePort --api-server-advertise-address=$(minikube ip -p minikube) --apiserver-enable-basic-auth=true --apiserver-enable-token-auth=true --apiserver-arg-overrides="
  --anonymous-auth=true,--v=4" --dns-provider-config="./coredns-provider.conf"
  ```

Show the federation api and controller running on minikube

  ```console
  kubectl get pods --context=minikube -n federation-system
  ```

Check the federation controller logs

  ```console
  kubectl logs -l module=federation-controller-manager --context=minikube -n federation-system
  ```

###### 4.2 change client to the federation api & create the default namespace

  ```console
  kubectl config use-context myfed
  kubectl create ns default
  ```

###### 4.3 join the us & europe clusters to the federation controller

  ```console
  kubefed join us --host-cluster-context=minikube
  kubefed join europe --host-cluster-context=minikube
  ```

###### 4.4 label the clusters

These labels are not required but can be used by the cluster selector or OPA policies.

  ```console
  kubectl label cluster us environment=test location=us pciCompliant=false
  kubectl label cluster europe environment=prod location=europe pciCompliant=true
  ```

view the labels

  ```
  kubectl get clusters -L environment -L location -L pciCompliant
  ```

# 5 Launch APP and Explore

###### 5.1 Start a Federated APP

Inspect the contents of `hello.yaml`.
Then apply it to the federation API.

  ```console
  kubectl apply -f ./hello.yaml
  ```

check for objects in each cluster

  ```console
  kubectl get svc,rs,po,ing --context=us
  kubectl get svc,rs,po,ing --context=europe
  ```

check for dns records

  ```console
  kubectl run -it --rm --restart=Never --image=infoblox/dnstools:latest dnstools --context=minikube
  dnstools# nslookup hello.default.myfed.svc.myzone coredns-coredns.default
  Server:		10.0.0.10
  Address:	10.0.0.10#53

  Name:	hello.default.myfed.svc.myzone
  Address: 192.168.99.101
  Name:	hello.default.myfed.svc.myzone
  Address: 192.168.99.102
  ```

###### 5.2 Scale up

  ```
  kubectl scale rs/hello --replicas=3
  ```

Check that the total count is now 3 across the clusters

  ```console
  kubectl get rs,po --context=us
  kubectl get rs,po --context=europe
  ```

###### 5.2 Select only europe w/ the cluster-selector

Inspect the contents of `hello-europe.yaml`.
Notice the additional annotation for `federation.alpha.kubernetes.io/cluster-selector`. 
Then apply it to the federation API.

  ```console
  kubectl apply -f ./hello-europe.yaml
  ```

verify that both replicas are running in europe now

  ```console
  kubectl get rs,po --context=us
  kubectl get rs,po --context=europe
  ```

# Cleanup

Remove federation components from the minikube context and delete the other minikube clusters.

  ```console
  kubectl delete ns federation-system --context=minikube
  minikube stop -p minikube
  minikube delete -p us
  minikube delete -p europe
  ```

Optional: nuke minikube cluster

  ```console
  minikube delete -p minikube
  ```


