## Pre-req:

- **[AWS CLI](https://aws.amazon.com/cli/)** (`aws`) v1.24.2 or higher (**[python](https://www.python.org/)** 3.8 installed via **[pyenv](https://github.com/pyenv/pyenv)** tested)
- **[Kubernetes client](https://kubernetes.io/docs/tasks/tools/#kubectl)** (`kubectl`) v1.24 or higher
- **[eksctl](https://eksctl.io/)**  v0.97 or higher
- **[Helm](https://helm.sh/)** [optional] v3.8.2 or higher
- **ACM** or **cert-manager (let's encrypt)**
    
    ## DNS and TLS Certificate
    
    If you wish to use **[DNS](https://en.wikipedia.org/wiki/Domain_Name_System)** records, you need to have a registered public domain on **[Amazon Route53](https://aws.amazon.com/route53/)**. We’ll use `mycompany.com.` as a fictional domain name for purposes of this tutorial. Replace this with your registered domain name.
    
    When created, you verify the hosted zone:
    
    ```
    aws route53 list-hosted-zones
    ```
    
    For secure traffic with **[TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security)**, you’ll need to have registered a public wildcard certificate matching the domain name with **[AWS Certificate Manager](https://aws.amazon.com/certificate-manager/)**, e.g. `*.mycompany.com`.
    
    Once created you can verify the certificate using:
    
    ```
    aws acm list-certificates--region us-west-2
    ```
    
    Note that unlike **[Route53](https://aws.amazon.com/route53/)**
     that is global, certificates are configured per region, so unless you 
    search in the region where you created the certificate, it will not be 
    listed.
    

Setup your env

```bash
export EKS_CLUSTER_NAME="my-demo-cluster"
export EKS_CLUSTER_REGION="us-west-2"
export KUBECONFIG=$HOME/kubeconfigs/demo-cluster-config.yaml
export EXTERNALDNS_NS="kube-addons"
export INGRESSNGINX_NS="kube-addons"
export NGINXDEMO_NS="demo"
export DOMAIN_NAME="thebrewery.xyz"
export ZONE_ID="Z0XXXXX"
```


## Part 1: Setup

```bash
$ eksctl create cluster \                                                                                               
  -f $HOME/eksctl_scripts/demo_cluster.yaml \
  --kubeconfig=$HOME/kubeconfigs/demo-cluster-config.yaml
```

```bash
$ k create secret generic --from-literal=aws_secret_key=xxxx  aws-creds
```

Adding Ingress Support

Now we can install the Ingress resource using the Ingress-Nginx controller. Under the hood, the controller is running openresty (NGINX + LuaJIT) reverse proxy and is exposed through a service LoadBalancer, which on EKS is ELBv1. This allows use through ACM to terminate the TLS certificate.

Before we begin you will need to get ACM ARN you create from your domain, which looks something like:

arn:aws:acm:us-west-2:XXXXXXXXXXXX:certificate/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX

For reference of what values to change:
- https://kubernetes.github.io/ingress-nginx/deploy/#network-load-balancer-nlb

```bash
$ k apply -f ingress_nginx_deploy.yaml
```

## Part 2: Create application

```bash
$ k apply -f hello-k8s.yaml
```

---

## Using cert-manager

Cert-Manager is an add-on utility for your Kubernetes Cluster, it automates the creating, issuing, and providing of certificates within your clusters. It provides a set of custom resources(YAML configuration) to issue certificates and attach them to services.

One of the most common use cases is securing web apps and APIs with SSL certificates from Let’s Encrypt. The following will guide you to add Cert-Manager to your EKS or Kubernetes cluster and set up a Let’s Encrypt certificate issuer through the YAML configuration file, and acquire and expose a certificate for Pods and Services exposed through your Nginx Ingress.

Install Cert-Manager

```bash
$ k apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.0/cert-manager.yaml
```

Install ClusterIssuer

```bash
$ k create secret generic --from-literal=secret-access-key=xxxx acme-route53 -n cert-manager
$ k apply -f clusterissuer.yaml

```

Test an application

```bash
$ k create ns test
$ k apply -f nginx-test.yaml -n test
```

For more info: https://cert-manager.io/docs/usage/ingress/#how-it-works
