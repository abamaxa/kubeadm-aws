## Cheap Kubernetes cluster on AWS with kubeadm

[Forked from...](https://github.com/cablespaghetti/kubeadm-aws)

This repository contains a bunch of Bash and Terraform code which provisions a single master Kubernetes cluster on AWS using spot instances. I have updated these scripts to work with kubernetes 1.16.4 and helm 2.14.1 but they are a not yet 100% reliable - sometimes
the master fails to initialize.

Current features:

* Automatic backup and recovery. So if your master gets terminated, when the replacement is provisioned by AWS it will pick up where the old one left off without you doing anything. 😁
* Completely automated provisioning through Terraform and Bash.
* Variables for many things including number of workers (provisioned using an auto-scaling group) and EC2 instance type.
* Helm (currently v2.14.1)
* Kubernetes v1.16.4.
* Supports EBS storage based images e.g. t3.medium.
* [External DNS](https://github.com/kubernetes-incubator/external-dns) and [Nginx Ingress](https://github.com/kubernetes/ingress-nginx) as a cheap ELB alternative, with [Cert Manager](https://github.com/jetstack/cert-manager) for TLS certificates via Let's Encrypt.
* Auto Scaling of worker nodes, if you enable the [Cluster AutoScaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler).
* Persistent Volumes using GP2 storage on EBS.

**This isn't designed for production (unless you're very brave) but I've found no stability issues so far for my personal development usage. However I have had instances where there is no available spot capacity for my chosen instance type in my Availability Zone which means you are without any nodes for a while...**

### Run it!

1. Clone the repo
2. [Install Terraform](https://www.terraform.io/intro/getting-started/install.html)
3. Make an SSH key locally in `~/.ssh/id_rsa.pub`
4. Run terraform plan: `terraform plan -var admin-cidr-blocks="<my-public-ip-address>/32"`
5. Build out infrastructure: `terraform apply -var admin-cidr-blocks="<my-public-ip-address>/32"`
6. SSH to K8S master and run something: `ssh ubuntu@$(terraform output master_dns) kubectl get no`
7. If you enabled cert-manager, the [Cert Manager Issuer](manifests/cert-manager-issuer.yaml.tmpl) for Let's Encrypt has been applied to the default namespace. You will also need to apply it to any other namespaces you want to obtain TLS certificates for.
8. Done!

Optional Variables:

* `min-worker-count` - The minimum size of the worker node Auto-Scaling Group (1 by default)
* `max-worker-count` - The maximum size of the worker node Auto-Scaling Group (1 by default)
* `region` - Which AWS region to use (eu-west-2 by default)
* `az` - Which AWS availability zone to use (a by default)
* `kubernetes-version` - Which Kubernetes/kubeadm version to install (1.16.4 by default)
* `master-instance-type` - Which EC2 instance type to use for the master node (m1.small by default)
* `master-spot-price` - The maximum spot bid for the master node ($0.01 by default)
* `worker-instance-type` - Which EC2 instance type to use for the worker nodes (m1.small by default)
* `worker-spot-price` - The maximum spot bid for worker nodes ($0.01 by default)
* `cluster-name` - Used for naming the created AWS resources (k8s by default)
* `backup-enabled` - Set to "0" to disable the automatic etcd backups (1 by default)
* `backup-cron-expression` - A cron expression to use for the automatic etcd backups (`*/15 * * * *` by default)
* `external-dns-enabled` - Set to "0" to disable ExternalDNS (1 by default) - Existing Route 53 Domain required
* `nginx-ingress-enabled` - Set to "1" to enable Nginx Ingress (0 by default)
* `nginx-ingress-domain` - The DNS name to map to Nginx Ingress using External DNS ("" by default)
* `cert-manager-enabled` - Set to "1" to enable Cert Manager (0 by default)
* `cert-manager-email` - The email address to use for Let's Encrypt certificate requests ("" by default)
* `cluster-autoscaler-enabled` - Set to "1" to enable the cluster autoscaler (0 by default)
* `k8stoken` - Override the automatically generated cluster bootstrap token

### Examples
* [Nginx deployment](examples/nginx.yaml)

### Ingress Notes

As hinted above, this uses Nginx Ingress as an alternative to a Load Balancer. This is done by exposing ports 443 and 80 directly on each of the nodes (Workers and the Master) using a NodePort type Service. Unfortunately External DNS doesn't seem to work with Nginx Ingress when you expose it in this way, so I've had to just map a single DNS name (using the nginx-ingress-domain variable) to the NodePort service itself. External DNS will keep that entry up to date with the IPs of the nodes in the cluster; you will then have to manually add CNAME entries for your individual services. Not the most secure way of exposing services, but it's secure enough for my purposes.

### Note about the license

I am not associated with UPMC Enterprises, but because this project started off as a fork of their code I am required to leave their license in place. However this is still Open Source and so you are free to do more-or-less whatever you want with the contents of this repository.

