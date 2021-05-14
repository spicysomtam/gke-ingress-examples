# Introduction

Experiment with ingress on Google Cloud Platform (gcp) Google Kubernetes Engine (gke).

How good is the gke (gke) ingress controller? How do we deploy it? How does it compare to standard ingress controllers (eg ingress-nginx), how do we install ingress-nginx in a gke kubernetes (k8s) cluster? Lets try these out.

We need a gke cluster to test this out, so we can either go to the web based Cloud console and build one there, or create one from the command line using `gcloud` (you can change or exclude some of these args to fit your needs; I had to get node quotas increased 8 to 20 to spin up gke in a zone):
```
gcloud container clusters create "my-first-cluster-1" \
  --zone "europe-west4-c" \
  --cluster-version "1.20.5-gke.2000" \
  --release-channel "rapid" \
  --machine-type "g1-small" \
  --node-locations "europe-west4-c"
```

Then setup your kube config for access to this cluster:
```
gcloud container clusters get-credentials my-first-cluster-1 --zone europe-west4-c
```

# GKE ingress controller

gke has its own ingress controller called gke (surprisingly). How do we implement this?

By default, gke ingress will create a load balancer for each ingress, which is wasteful. Its also quite slow creating the load balancer.

Examples taken from [Google official docs](https://cloud.google.com/kubernetes-engine/docs/how-to/load-balance-ingress).

There is a way of having multiple hosts behind a single gke ingress load balancer; take a look at the [gke ingress definition](ingress-gke/gke-ingress.yaml).

Now lets deploy some services and the ingress (in the `default` namespace):
```
kubectl apply -f ingress-gke/
```

Test access; IP is the load balancer IP created by the gke ingress; check Load Balancers under Network Services in the web console:

```
$ curl -H host:ng2.example.com http://34.91.10.180/
Hello Kubernetes!

$ curl -H host:ng1.example.com http://34.91.10.180/
Hello, world!
Version: 2.0.0
Hostname: hello-world-55c6fc8855-z8zxg
```

Pros of the gke ingress:
* Uses gke ingress; no need to deploy own ingress.
* Simple to implement.

Cons of the gke ingress:
* Time consuming for gke to deploy.
* Lacks the flexibility of other ingress controllers that use a TCP load balancer.
* Load balancer is layer 7 and somewhat managed by the gcp infra (eg complex) rather than k8s.
* ingress-nginx (and other ingress controllers) are much better (believe me!).

Lets cleanup after testing:
```
kubectl delete -f ingress-gke/
```

Alternative? Deploy ingress-nginx (default kubernetes one) or others with a tcp load balancer.

# ingress-nginx ingress controller

You could use other popular ingress controllers (eg haproxy, traefik, nginx inc, etc). However we will stick with the standard Kubernetes ingress, which is not installed by default in a gke cluster. Thus we will need to install it:

As per official `ingress-nginx` docs:
```
kubectl create clusterrolebinding cluster-admin-binding  \
   --clusterrole cluster-admin \
   --user $(gcloud config get-value account)
```

Add the ingress-nginx controller; we use `helm` as it does the setup correctly as a deployment with a single pod using a load balancer:
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --create-namespace \
  --namespace ingress-nginx
```

This creates the `ingress-nginx` ingress controller in a namespace of the same name. The deploy created a deployment with a single pod, and a service of type `LoadBalancer`, which allows all k8s nodes to receive traffic for the ingress. This setup is fine for testing and small deployments, although for higher volumes you may want to switch to a `DaemonSet` deployment, which will create a ingress pod on each k8s node, and provides greater throughput (and resilience).

Now lets deploy some services and the ingress (in the `default` namespace):
```
kubectl apply -f ingress-nginx/
```
Test access (similar to earlier); IP is the load balancer for the service created for ingress-nginx; check services in the `ingress-nginx` namespace.

```
$ curl -H host:ng2.example.com http://34.90.25.34/
Hello Kubernetes!

$ curl -H host:ng1.example.com http://34.90.25.34/
Hello, world!
Version: 2.0.0
Hostname: hello-world-55c6fc8855-z8zxg
```

Lets cleanup after testing:
```
kubectl delete -f ingress-nginx/
helm uninstall ingress-nginx -n ingress-nginx
```

Pros for this ingress controller:
* More functionality than the gke ingress.
* Single tcp layer load balancer can receive traffic for multiple dns names (and/or host paths).
* Quick deployment of additional ingress definitions; can define many to use the single load balancer.

Cons for this ingress controller:
* Additional work required for setup.

# Delete cluster after testing

```
gcloud container clusters delete my-first-cluster-1 --zone europe-west4-c
```
