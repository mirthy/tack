# terraform-aws-coreos-kubernetes

[![Gitter](https://badges.gitter.im/kz8s/tack.svg)](https://gitter.im/kz8s/tack?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)
[![Circle CI](https://circleci.com/gh/kz8s/tack.svg?style=svg)](https://circleci.com/gh/kz8s/tack)

Opinionated [Terraform](https://terraform.io) module for creating a Highly Available [Kubernetes](http://kubernetes.io) cluster running on
[CoreOS](https://coreos.com) (any channel) in an [AWS](https://aws.amazon.com)
Virtual Private Cloud ([VPC](https://aws.amazon.com/vpc/)). With prerequisites
installed `make all` will simply spin up a default cluster; and, since it is
based on Terraform, customization is much easier than
[CloudFormation](https://aws.amazon.com/cloudformation/).

The default configuration includes Kubernetes
[add-ons](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons):
DNS, Dashboard and UI.

## tl;dr
```bash
# prereqs
$ brew update && brew install awscli cfssl jq kubernetes-cli terraform

# build artifacts and deploy cluster
$ make all

# nodes
$ kubectl get nodes

# addons
$ kubectl get pods --namespace=kube-system

# verify dns - run after addons have fully loaded
$ kubectl exec busybox -- nslookup kubernetes

# open dashboard
$ make dashboard

# obliterate the cluster and all artifacts
$ make clean
```

## Features
* TLS certificate generation

### AWS
* [EC2 Key Pair](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)
creation
* AWS VPC Public and Private subnets
* IAM protected S3 bucket for asset (TLS and manifests) distribution
* Bastion Host
* Multi-AZ Auto-Scaling Worker Nodes
* [NAT Gateway](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-nat-gateway.html)

### CoreOS (1122.2.0, 1122.1.0, 1122.0.0)
* etcd DNS Discovery Bootstrap
* kubelet runs under rkt (using CoreOS recommended [Kubelet Wrapper Script](https://coreos.com/kubernetes/docs/latest/kubelet-wrapper.html))

### Kubernetes (v1.3.6)
* [Highly Available ApiServer Configuration](http://kubernetes.io/v1.1/docs/admin/high-availability.html)
* Service accounts enabled
* SkyDNS utilizing cluster's etcd

### Terraform (v0.7.3)
* CoreOS AMI sourcing
* Terraform Pattern Modules

## Prerequisites
* [AWS Command Line Interface](http://aws.amazon.com/documentation/cli/)
* [CFSSL: CloudFlare's PKI and TLS toolkit](https://cfssl.org/)
* [jq](https://stedolan.github.io/jq/)
* [kubectl](http://kubernetes.io/v1.1/docs/user-guide/kubectl-overview.html)
* [Terraform](https://www.terraform.io/)

Quick install prerequisites on Mac OS X with [Homebrew](http://brew.sh/):

```bash
$ brew update && brew install awscli cfssl jq kubernetes-cli terraform
```

Tested with prerequisite versions:

```bash
$ aws --version
aws-cli/1.10.58 Python/2.7.10 Darwin/15.6.0 botocore/1.4.48

$ cfssl version
Version: 1.2.0
Revision: dev
Runtime: go1.6.2

$ jq --version
jq-1.5

$ kubectl version --client
Client Version: version.Info{Major:"1", Minor:"3", GitVersion:"v1.3.5+b0deb2e", GitCommit:"b0deb2eb8f4037421077f77cb163dbb4c0a2a9f5", GitTreeState:"not a git tree", BuildDate:"2016-08-21T12:21:36Z", GoVersion:"go1.7", Compiler:"gc", Platform:"darwin/amd64"}

$ terraform --version
Terraform v0.7.3
```

## Launch Cluster

`make all` will create:
- AWS Key Pair (PEM file)
- client and server TLS assets
- s3 bucket for TLS assets (secured by IAM roles for master and worker nodes)
- AWS VPC with private and public subnets
- Route 53 internal zone for VPC
- Etcd cluster bootstrapped from Route 53
- High Availability Kubernetes configuration (masters running on etcd nodes)
- Autoscaling worker node group across subnets in selected region
- kube-system namespace and addons: DNS, UI, Dashboard

```bash
$ make all
```

To open dashboard:

```bash
$ make dashboard
```

To destroy, remove and generally undo everything:

```
$ make clean
```

`make all` and `make clean` should be idempotent - should an error occur simply try running
the command again and things should recover from that point.

## How Tack works

### Tack Phases

Tack works in three phases:

1. Pre-Terraform
2. Terraform
3. Post-Terraform

#### Pre-Terraform
The purpose of this phase is to prep the environment for Terraform execution. Some tasks are
hard or messy to do in Terraform - a little prep work can go a long way here. Determining
the CoreOS AMI for a given region, channel and VM Type for instance is easy enough to do
with a simple shell script.

#### Terraform
Terraform does the heavy lifting of resource creation and sequencing. Tack uses local
modules to partition the work in a logical way. Although it is of course possible to do all
of the Terraform work in a single `.tf` file or collection of `.tf` files, it becomes
unwieldy quickly and impossible to debug. Breaking the work into local modules makes the
flow much easier to follow and provides the basis for composing variable solutions down the track - for example converting the worker Auto Scaling Group to use spot instances.

#### Post-Terraform
Once the infrastructure has been configured and instantiated it will take some type for it
to settle. Waiting for the 'master' ELB to become healthy is an example of this.  

### Components

Like many great tools, _tack_ has started out as a collection of scripts, makefiles and other tools. As _tack_ matures and patterns crystalize it will evolve to a Terraform plugin and perhaps a Go-based cli tool for 'init-ing' new cluster configurations. The tooling will compose Terraform modules into a solution based on user preferences - think `npm init` or better yet [yeoman](http://yeoman.io/).

#### TLS Certificates

* [etcd coreos cloudint](https://github.com/coreos/coreos-cloudinit/blob/master/config/etcd.go)

```bash
$ curl --cacert /etc/kubernetes/ssl/ca.pem  https://etcd1.k8s:2379/version
$ openssl x509 -text -noout -in /etc/kubernetes/ssl/ca.pem
$ openssl x509 -text -noout -in /etc/kubernetes/ssl/k8s-etcd.pem
```

#### ElasticSearch and Kibana

To access Elasticseach and Kibana first start `kubectl proxy`.

```bash
$ kubectl proxy
Starting to serve on localhost:8001
```

* [http://localhost:8001/api/v1/proxy/namespaces/kube-system/services/elasticsearch-logging ](http://localhost:8001/api/v1/proxy/namespaces/kube-system/services/elasticsearch-logging)
* [http://localhost:8001/api/v1/proxy/namespaces/kube-system/services/kibana-logging](http://localhost:8001/api/v1/proxy/namespaces/kube-system/services/kibana-logging)

## FAQs

* [Create an etcd cluster with more than 3 instances](https://github.com/kz8s/tack/wiki/How-to:-change-etcd-cluster-size)

## Advanced Features and Configuration

### Using an Existing VPC

If you have an existing VPC you'd like to deploy a cluster into, there is an option for this with _tack_.

#### Constraints
* You will need to allocate 3 static IPs for the etcd servers - Just choose 3 IPs in a subnet that's not being used
* Your VPC has to have private and public subnets (for now)
* You will need to know the following information:
  * VPC CIDR Range (e.g. 192.168.0.0/16)
  * VPC Id (e.g. vpc-abc123)
  * VPC Internet Gateway Id (e.g. igw-123bbd)
  * VPC Public Subnet Ids (e.g. subnet-xyz123,subnet-zyx123)
  * VPC Private Subnet Ids (e.g. subnet-lmn123,subnet-opq123)

#### Enabling Existing VPC Support
* Edit vpc-existing.tfvars
  * Uncomment the blocks with variables and fill in the missing information
* Edit modules_override.tf - This uses the [overrides feature from Terraform](https://www.terraform.io/docs/configuration/override.html)
  * Uncomment the vpc module, this will override the reference to the regular VPC module and instead use the stub vpc-existing module which just pulls in the variables from vpc-existing.tfvars
* Edit the Makefile as necessary for POD_IP_RANGE, K8S_SERVICE_IP_RANGE, etc to match what you need (e.g. avoid collisions with existing IP ranges in your VPC or extended infrastructure)

#### Testing Existing VPC Support from Scratch
In order to test existing VPC support, we need to generate a VPC and then try the overrides with it. After that we can clean it all up.  These instructions are meant for someone wanting to ensure that the _tack_ existing VPC code works properly.
* Run `make all` to generate a VPC with Terraform
* Edit terraform.tfstate
  * Search for the VPC block and cut it out and save it somewhere.  Look for "path": ["root","vpc"]
* Run `make clean` to remove everything but the VPC and associated networking (we preserved it in the previous step)
* Edit as per instructions above
* Run `make all` to test out using an existing VPC
* Cleaning up:
  * Re-insert the VPC block into terraform.tfstate
  * Run `make clean` to clean up everything

#### Additional Configuration
* You probably want to [tag your subnets](https://github.com/kubernetes/kubernetes/blob/master/pkg/cloudprovider/providers/aws/aws.go#66) for internal/external load balancers

## Inspiration
* [Code examples to create CoreOS cluster on AWS with Terraform](https://github.com/xuwang/aws-terraform) by [xuwang](https://github.com/xuwang)
* [kaws: tool for deploying multiple Kubernetes clusters](https://github.com/InQuicker/kaws)
* [Kubernetes on CoreOS](https://github.com/coreos/coreos-kubernetes)
* [Terraform Infrastructure Design Patterns](https://www.opencredo.com/2015/09/14/terraform-infrastructure-design-patterns/) by [Bart Spaans](https://www.opencredo.com/author/bart/)
* [The infrastructure that runs Brandform](https://github.com/brandfolder/infrastructure)


## Other Terraform Projects
* [bakins/kubernetes-coreos-terraform](https://github.com/bakins/kubernetes-coreos-terraform)
* [bobtfish/terraform-aws-coreos-kubernates-cluster](https://github.com/bobtfish/terraform-aws-coreos-kubernates-cluster)
* [chiefy/tf-aws-kubernetes](https://github.com/chiefy/tf-aws-kubernetes)
* [cihangir/terraform-aws-kubernetes](https://github.com/cihangir/terraform-aws-kubernetes)
* [ericandrewlewis/kubernetes-via-terraform](https://github.com/ericandrewlewis/kubernetes-via-terraform)
* [funkymonkeymonk/terraform-demo](https://github.com/funkymonkeymonk/terraform-demo)
* [kelseyhightower/kubestack](https://github.com/kelseyhightower/kubestack)
* [samsung-cnct/kraken](https://github.com/samsung-cnct/kraken)
* [wearemakery/kubestack-aws](https://github.com/wearemakery/kubestack-aws)
* [xuwang/aws-terraform](https://github.com/xuwang/aws-terraform)

## References
* [CFSSL: CloudFlare's PKI and TLS toolkit](https://cfssl.org/)
* [CoreOS - Mounting Storage](https://coreos.com/os/docs/latest/mounting-storage.html)
* [Deploying CoreOS cluster with etcd secured by TLS/SSL](http://blog.skrobul.com/securing_etcd_with_tls/)
* [etcd dns discovery bootstrap](https://coreos.com/etcd/docs/latest/clustering.html#dns-discovery)
* [Generate EC2 Key Pair](https://github.com/xuwang/aws-terraform/blob/master/scripts/aws-keypair.sh)
* [Generate self-signed certificates](https://coreos.com/os/docs/latest/generate-self-signed-certificates.html)
* [Makefile `help` target](https://gist.github.com/rcmachado/af3db315e31383502660)
* [Peeking under the hood of Kubernetes on AWS](https://github.com/kubernetes/kubernetes/blob/release-1.2/docs/design/aws_under_the_hood.md)
* [Self documenting Makefile](https://gist.github.com/prwhite/8168133)
* [Setting up etcd to run in production](https://github.com/kelseyhightower/etcd-production-setup)
* [ssl artifact generation](https://github.com/coreos/coreos-kubernetes/tree/master/lib)
