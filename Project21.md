# STEP 0-INSTALL CLIENT TOOLS BEFORE BOOTSTRAPPING THE CLUSTER.

First, you will need some client tools installed and configurations made on your client workstation:

awscli – is a unified tool to manage your AWS services
kubectl – this command line utility will be your main control tool to manage your K8s cluster. You will use this tool so many times, so you will be able to type ‘kubetcl’ on your keyboard with a speed of light. You can always make a shortcut (alias) to just one character ‘k’. Also, add this extremely useful official kubectl Cheat Sheet to your bookmarks, it has examples of the most used ‘kubectl’ commands.
cfssl – an open source toolkit for everything TLS/SSL from Cloudflare
cfssljson – a program, which takes the JSON output from the cfssl and writes certificates, keys, CSRs, and bundles to disk.
Install and configure AWS CLI
Configure AWS CLI to access all AWS services used, for this you need to have a user with programmatic access keys configured in AWS Identity and Access Management (IAM):

Generate access keys and store them in a safe place.

On your local workstation download and install the latest version of AWS CLI

To configure your AWS CLI – run your shell (or cmd if using Windows) and run:

[AWS CLI](./images/AWS-CLI-Config.PNG)

Test your AWS CLI by running and check if you can see VPC details:

`aws ec2 describe-vpcs`

Install kubectl
Kubernetes cluster has a Web API that can receive HTTP/HTTPS requests, but it is quite cumbersome to curl an API each and every time you need to send some command, so kubectl command tool was developed to ease a K8s administrator’s life.

With this tool you can easily interact with Kubernetes to deploy applications, inspect and manage cluster resources, view logs and perform many more administrative operations.

Installing kubectl

Linux Or Windows using Gitbash or similar tool

Download the binary:

`wget https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl`

[Download Binary](./images/binary-dwnld.PNG)

Make it executable:

`chmod +x kubectl`

Move to the Bin directory:

`sudo mv kubectl /usr/local/bin/`

[Download Binary](./images/chmod-sudo.PNG)

Verify that kubectl version 1.21.0 or higher is installed:

`kubectl version --client`  

[Kubectl Version](./images/kube-vers-output.PNG)

Install CFSSL and CFSSLJSON
cfssl is an open source tool by Cloudflare used to setup a Public Key Infrastructure (PKI Infrastructure) for generating, signing and bundling TLS certificates. In previous projects you have experienced the use of Letsencrypt for the similar use case. Here, cfssl will be configured as a Certificate Authority which will issue the certificates required to spin up a Kubernetes cluster.

Download, install and verify successful installation of cfssl and cfssljson:

Install CFSSL and CFSSLJSON-linux:

`wget https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl-bundle_1.6.4_linux_amd64` 

`mv cfssl-bundle_1.6.4_linux_amd64 cfssl` 

`chmod +x cfssl`

`sudo mv cfssl /usr/local/bin/`


<!-- `wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssl \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssljson`   -->

[CFSSL & CFSSLJSON](./images/cfss-install.PNG)

[CFSSL Version](./images/cfssl-vers.PNG)

[CFSSLJson Version](./images/cfssljson-vers.PNG)

- Kubectl: Kubernetes command-line tool, allows you to run commands against Kubernetes clusters. You can use kubectl to deploy applications, inspect and manage cluster resources, and view logs.

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y  kubectl
sudo apt-mark hold kubectl

kubectl version --client.


# AWS CLOUD RESOURCES FOR KUBERNETES CLUSTER.

As we already know, we need some machines to run the control plane and the worker nodes. In this section, you will provision EC2 Instances required to run your K8s cluster. You can use Terraform for this. But it is highly recommended to start out first with manual provisioning using awscli and have thorough knowledge about the whole setup. After that, you can destroy the entire project and start all over again using Terraform. This manual approach will solidify your skills and give you the opportunity to face more challenges.

To use AWSCLI, run:

`aws configure`

Step 1 – Configure Network Infrastructure
Virtual Private Cloud – VPC

1. Create a directory named k8s-cluster-from-ground-up

2. Create a VPC and store the ID as a variable:

`VPC_ID=$(aws ec2 create-vpc \
--cidr-block 172.31.0.0/16 \
--output text --query 'Vpc.VpcId'
)`

3. Tag the VPC so that it is named:

`aws ec2 create-tags \
  --resources ${VPC_ID} \
  --tags Key=Name,Value=${NAME}`

Domain Name System – DNS

4. Enable DNS support for your VPC:

`aws ec2 modify-vpc-attribute --vpc-id vpc-00cb9af694545376e --enable-dns-support '{"Value": true}'`

5. Enable DNS support for hostnames:

`aws ec2 modify-vpc-attribute --vpc-id vpc-00cb9af694545376e --enable-dns-hostnames '{"Value": true}'`

[VPC Config](./images/awsvpc-support-enable.PNG)

AWS Region

6. Set the required region:

`AWS_REGION=eu-central-1`

Dynamic Host Configuration Protocol – DHCP

7. Dynamic Host Configuration Protocol (DHCP) is a network management protocol used on Internet Protocol networks for automatically assigning IP addresses and other communication parameters to devices connected to the network using a client–server architecture.

AWS automatically creates and associates a DHCP option set for your Amazon VPC upon creation and sets two options: domain-name-servers (defaults to AmazonProvidedDNS) and domain-name (defaults to the domain name for your set region). AmazonProvidedDNS is an Amazon Domain Name System (DNS) server, and this option enables DNS for instances to communicate using DNS names.

By default EC2 instances have fully qualified names like ip-172-50-197-106.eu-central-1.compute.internal. But you can set your own configuration using an example below.

`DHCP_OPTION_SET_ID=$(aws ec2 create-dhcp-options \
  --dhcp-configuration \
    "Key=domain-name,Values=$AWS_REGION.compute.internal" \
    "Key=domain-name-servers,Values=AmazonProvidedDNS" \
  --output text --query 'DhcpOptions.DhcpOptionsId')`
