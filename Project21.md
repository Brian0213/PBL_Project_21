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

`AWS_REGION=eu-west-1`

Dynamic Host Configuration Protocol – DHCP

7. Dynamic Host Configuration Protocol (DHCP) is a network management protocol used on Internet Protocol networks for automatically assigning IP addresses and other communication parameters to devices connected to the network using a client–server architecture.

AWS automatically creates and associates a DHCP option set for your Amazon VPC upon creation and sets two options: domain-name-servers (defaults to AmazonProvidedDNS) and domain-name (defaults to the domain name for your set region). AmazonProvidedDNS is an Amazon Domain Name System (DNS) server, and this option enables DNS for instances to communicate using DNS names.

By default EC2 instances have fully qualified names like ip-172-50-197-106.eu-central-1.compute.internal. But you can set your own configuration using an example below.

`DHCP_OPTION_SET_ID=$(aws ec2 create-dhcp-options \
  --dhcp-configuration \
    "Key=domain-name,Values=$AWS_REGION.compute.internal" \
    "Key=domain-name-servers,Values=AmazonProvidedDNS" \
  --output text --query 'DhcpOptions.DhcpOptionsId')`

8. Tag the DHCP Option set:

`aws ec2 create-tags \
  --resources ${DHCP_OPTION_SET_ID} \
  --tags Key=Name,Value=${k8s-cluster-from-ground-up}`

[VPC Config](./images/dhcp-tag.PNG)

9. Associate the DHCP Option set with the VPC:

`aws ec2 associate-dhcp-options --dhcp-options-id ${DHCP_OPTION_SET_ID} --vpc-id vpc-00cb9af694545376e`

[DHCP Associate VPC](./images/dhcp-associate-vpc.PNG)

SUBNET

10. Create the Subnet:

`SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id vpc-00cb9af694545376e \
  --cidr-block 172.31.0.0/24 \
  --output text --query 'Subnet.SubnetId')`

`aws ec2 create-tags \
  --resources ${SUBNET_ID} \
  --tags Key=Name,Value=${k8s-cluster-from-ground-up}`

[Subnet VPC](./images/subnet-vpc.PNG)

Internet Gateway – IGW

11. Create the Internet Gateway and attach it to the VPC:

`INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway \
  --output text --query 'InternetGateway.InternetGatewayId')
aws ec2 create-tags \
  --resources ${INTERNET_GATEWAY_ID} \
  --tags Key=Name,Value=${k8s-cluster-from-ground-up}`

`aws ec2 attach-internet-gateway \
  --internet-gateway-id ${INTERNET_GATEWAY_ID} \
  --vpc-id vpc-00cb9af694545376e`

[Internet Gateway](./images/igw-config.PNG)

Route Tables

12. Create route tables, associate the route table to subnet, and create a route to allow external traffic to the Internet through the Internet Gateway:

`ROUTE_TABLE_ID=$(aws ec2 create-route-table \
  --vpc-id vpc-00cb9af694545376e \
  --output text --query 'RouteTable.RouteTableId')`

`aws ec2 create-tags \
  --resources ${ROUTE_TABLE_ID} \
  --tags Key=Name,Value=${k8s-cluster-from-ground-up}`

`aws ec2 associate-route-table \
  --route-table-id ${ROUTE_TABLE_ID} \
  --subnet-id subnet-0b34a94fac58256fc`

`aws ec2 create-route \
  --route-table-id ${ROUTE_TABLE_ID} \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-08459e721f03a7ae6`

[Route Tables](./images/rtb-config1.PNG)

[Route Tables](./images/rtb-config2.PNG)

[Route Tables](./images/aws-rtb-output.PNG)

SECURITY GROUPS

13. Configure security groups:

# Create the security group and store its ID in a variable

`SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name ${k8s-cluster-from-ground-up} \
  --description "Kubernetes cluster security group" \
  --vpc-id vpc-00cb9af694545376e \
  --output text --query 'GroupId')`

# Create the NAME tag for the security group

`aws ec2 create-tags \
  --resources sg-03892885a0637b5f3 \
  --tags Key=Name,Value=${k8s-cluster-from-ground-up}`

# Create Inbound traffic for all communication within the subnet to connect on ports used by the master node(s)

`aws ec2 authorize-security-group-ingress \
    --group-id sg-03892885a0637b5f3 \
    --ip-permissions IpProtocol=tcp,FromPort=2379,ToPort=2380,IpRanges='[{CidrIp=172.31.0.0/24}]'`

# Create Inbound traffic for all communication within the subnet to connect on ports used by the worker nodes

`aws ec2 authorize-security-group-ingress \
    --group-id sg-03892885a0637b5f3 \
    --ip-permissions IpProtocol=tcp,FromPort=30000,ToPort=32767,IpRanges='[{CidrIp=172.31.0.0/24}]'`

# Create inbound traffic to allow connections to the Kubernetes API Server listening on port 6443

`aws ec2 authorize-security-group-ingress \
  --group-id sg-03892885a0637b5f3 \
  --protocol tcp \
  --port 6443 \
  --cidr 0.0.0.0/0`

# Create Inbound traffic for SSH from anywhere (Do not do this in production. Limit access ONLY to IPs or CIDR that MUST connect)

`aws ec2 authorize-security-group-ingress \
  --group-id sg-03892885a0637b5f3 \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0`

# Create ICMP ingress for all types

`aws ec2 authorize-security-group-ingress \
  --group-id sg-03892885a0637b5f3 \
  --protocol icmp \
  --port -1 \
  --cidr 0.0.0.0/0`

[Security Group](./images/aws-secgrp1.PNG)

[Security Group](./images/aws-secgrp2.PNG)

Network Load Balancer

14. Create a network Load balancer:

`LOAD_BALANCER_ARN=$(aws elbv2 create-load-balancer \
--name ${k8s-cluster-from-ground-up} \
--subnets subnet-0b34a94fac58256fc \
--scheme internet-facing \
--type network \
--output text --query 'LoadBalancers[].LoadBalancerArn')`

[Load Balancer](./images/aws-loadbalan.PNG)

Tagret Group

15. Create a target group: (For now it will be unhealthy because there are no real targets yet.)

`TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
  --name ${k8s-cluster-from-ground-up} \
  --protocol TCP \
  --port 6443 \
  --vpc-id vpc-00cb9af694545376e \
  --target-type ip \
  --output text --query 'TargetGroups[].TargetGroupArn')`

[Target Group](./images/aws-targrp.PNG)

16. Register targets: (Just like above, no real targets. You will just put the IP addresses so that, when the nodes become available, they will be used as targets.)

`aws elbv2 register-targets \
  --target-group-arn ${TARGET_GROUP_ARN} \
  --targets Id=172.31.0.1{0,1,2}`

17. Create a listener to listen for requests and forward to the target nodes on TCP port 6443:

`aws elbv2 create-listener \
--load-balancer-arn ${LOAD_BALANCER_ARN} \
--protocol TCP \
--port 6443 \
--default-actions Type=forward,TargetGroupArn=${TARGET_GROUP_ARN} \
--output text --query 'Listeners[].ListenerArn'`

[Target Group](./images/lister-cli.PNG)

[Target Group](./images/targrp-loadbalan-asso.PNG)

K8s Public Address

18. Get the Kubernetes Public address:

`KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
--load-balancer-arns ${LOAD_BALANCER_ARN} \
--output text --query 'LoadBalancers[].DNSName')`


Step 2 – Create Compute Resources
AMI

1. Get an image to create EC2 instances:

`IMAGE_ID=$(aws ec2 describe-images --owners 099720109477 \
  --filters \
  'Name=root-device-type,Values=ebs' \
  'Name=architecture,Values=x86_64' \
  'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-xenial-16.04-amd64-server-*' \
  | jq -r '.Images|sort_by(.Name)[-1]|.ImageId')`

`echo $IMAGE_ID`

[AMI Image](./images/ami-image.PNG)

[Compute 1](./images/step2.1.output.PNG)

SSH key-pair

2. Create SSH Key-Pair:

`mkdir -p ssh`

`aws ec2 create-key-pair \ --key-name ${k8s-cluster-from-ground-up} \ --output text --query 'KeyMaterial' \ > ssh/${k8s-cluster-from-ground-up}.id_rsa
chmod 600 ssh/${k8s-cluster-from-ground-up}.id_rsa`

[SSH Create](./images/ssh-keypair-output.PNG)

EC2 Instances for Controle Plane (Master Nodes)

3. Create 3 Master nodes: Note – Using t2.micro instead of t2.small as t2.micro is covered by AWS free tier:

for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ami-0f29c8402f8cce65c \
    --count 1 \
    --key-name ${NAME} \
    --security-group-ids sg-03892885a0637b5f3 \
    --instance-type t2.micro \
    --private-ip-address 172.31.0.1${i} \
    --user-data "name=master-${i}" \
    --subnet-id subnet-0b34a94fac58256fc \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute \
    --instance-id ${instance_id} \
    --no-source-dest-check
  aws ec2 create-tags \
    --resources ${instance_id} \
    --tags "Key=Name,Value=${k8s-cluster-from-ground-up}-master-${i}"
 done

EC2 Instances for Worker Nodes

1. Create 3 worker nodes:

for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ami-0f29c8402f8cce65c \
    --count 1 \
    --key-name ${NAME} \
    --security-group-ids sg-03892885a0637b5f3 \
    --instance-type t2.micro \
    --private-ip-address 172.31.0.2${i} \
    --user-data "name=worker-${i}|pod-cidr=172.20.${i}.0/24" \
    --subnet-id subnet-0b34a94fac58256fc \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute \
    --instance-id ${instance_id} \
    --no-source-dest-check
  aws ec2 create-tags \
    --resources ${instance_id} \
    --tags "Key=Name,Value=${k8s-cluster-from-ground-up}-worker-${i}"
done

STEP 3 PREPARE THE SELF-SIGNED CERTIFICATE AUTHORITY AND GENERATE TLS CERTIFICATES
Step 3 Prepare The Self-Signed Certificate Authority And Generate TLS Certificates
The following components running on the Master node will require TLS certificates.

kube-controller-manager
kube-scheduler
etcd
kube-apiserver
The following components running on the Worker nodes will require TLS certificates.

kubelet
kube-proxy
Therefore, you will provision a PKI Infrastructure using cfssl which will have a Certificate Authority. The CA will then generate certificates for all the individual components.

Self-Signed Root Certificate Authority (CA)

Here, you will provision a CA that will be used to sign additional TLS certificates.

Create a directory and cd into it:

`mkdir ca-authority && cd ca-authority`

Generate the CA configuration file, Root Certificate, and Private key:

`{

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "Kubernetes",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}`

[Output](./images/output.PNG)

List the directory to see the created files:

`ls -ltr`

[CA configuration file, Root Certificate, and Private key](./images/root-prvkey-config.PNG)

The 3 important files here are:

ca.pem – The Root Certificate
ca-key.pem – The Private Key
ca.csr – The Certificate Signing Request
Generating TLS Certificates For Client and Server

You will need to provision Client/Server certificates for all the components. It is a MUST to have encrypted communication within the cluster. Therefore, the server here are the master nodes running the api-server component. While the client is every other component that needs to communicate with the api-server.

Now we have a certificate for the Root CA, we can then begin to request more certificates which the different Kubernetes components, i.e. clients and server, will use to have encrypted communication.

Remember, the clients here refer to every other component that will communicate with the api-server. These are:

kube-controller-manager
kube-scheduler
etcd
kubelet
kube-proxy
Kubernetes Admin User
Let us begin with the Kubernetes API-Server Certificate and Private Key

The certificate for the Api-server must have IP addresses, DNS names, and a Load Balancer address included. Otherwise, you will have a lot of difficulties connecting to the api-server.

1. Generate the Certificate Signing Request (CSR), Private Key and the Certificate for the Kubernetes Master Nodes:

`{
cat > master-kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
   "hosts": [
   "127.0.0.1",
   "172.31.0.10",
   "172.31.0.11",
   "172.31.0.12",
   "ip-172-31-0-10",
   "ip-172-31-0-11",
   "ip-172-31-0-12",
   "ip-172-31-0-10.${AWS_REGION}.compute.internal",
   "ip-172-31-0-11.${AWS_REGION}.compute.internal",
   "ip-172-31-0-12.${AWS_REGION}.compute.internal",
   "${KUBERNETES_PUBLIC_ADDRESS}",
   "kubernetes",
   "kubernetes.default",
   "kubernetes.default.svc",
   "kubernetes.default.svc.cluster",
   "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "Kubernetes",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  master-kubernetes-csr.json | cfssljson -bare master-kubernetes
}`


[CA configuration file, Root Certificate, and Private key](./images/kube-csr-prvkey-cert.PNG)

Creating the other certificates: for the following Kubernetes components:

Scheduler Client Certificate
Kube Proxy Client Certificate
Controller Manager Client Certificate
Kubelet Client Certificates
K8s admin user Client Certificate

2. kube-scheduler Client Certificate and Private Key:

`{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:kube-scheduler",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}`

[kube-scheduler Client Certificate and Private Key](./images/kubesche-cert-prvkey.PNG)

If you see any warning message, it is safe to ignore it.

3. kube-proxy Client Certificate and Private Key:

`{

cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:node-proxier",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}`

[kube-proxy Client Certificate and Private Key](./images/kubeproxy-cert-prvkey.PNG)

4. kube-controller-manager Client Certificate and Private Key:

`{
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:kube-controller-manager",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}`

[kube-controller-manager Client Certificate and Private Key](./images/kubecontrmgr-cert-prvkey.PNG)

5. kubelet Client Certificate and Private Key

Similar to how you configured the api-server's certificate, Kubernetes requires that the hostname of each worker node is included in the client certificate.

Also, Kubernetes uses a special-purpose authorization mode called Node Authorizer, that specifically authorizes API requests made by kubelet services. In order to be authorized by the Node Authorizer, kubelets must use a credential that identifies them as being in the system:nodes group, with a username of system:node:<nodeName>. Notice the "CN": "system:node:${instance_hostname}", in the below code.

Therefore, the certificate to be created must comply to these requirements. In the below example, there are 3 worker nodes, hence we will use bash to loop through a list of the worker nodes’ hostnames, and based on each index, the respective Certificate Signing Request (CSR), private key and client certificates will be generated.

`for i in 0 1 2; do
  instance="${NAME}-worker-${i}"
  instance_hostname="ip-172-31-0-2${i}"
  cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance_hostname}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:nodes",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  internal_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PrivateIpAddress')

  cfssl gencert \
    -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -hostname=${instance_hostname},${external_ip},${internal_ip} \
    -profile=kubernetes \
    ${NAME}-worker-${i}-csr.json | cfssljson -bare ${NAME}-worker-${i}
done`

[kubelet Client Certificate and Private Key](./images/kubelet-cert-prvkey.PNG)

6. Finally, kubernetes admin user's Client Certificate and Private Key

`{
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:masters",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
}`

[kubernetes admin user's Client Certificate and Private Key](./images/kubeadminuser-cert-prvkey.PNG)

7. Actually, we are not done yet! :tired_face:
There is one more pair of certificate and private key we need to generate. That is for the Token Controller: a part of the Kubernetes Controller Manager kube-controller-manager responsible for generating and signing service account tokens which are used by pods or other resources to establish connectivity to the api-server. Read more about Service Accounts from the official documentation.

Alright, let us quickly create the last set of files, and we are done with PKIs

`{

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "Kubernetes",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
}`

[Token Controller](./images/token-controll.PNG)

STEP 4 – DISTRIBUTING THE CLIENT AND SERVER CERTIFICATES
Step 4 – Distributing the Client and Server Certificates
Now it is time to start sending all the client and server certificates to their respective instances.

Let us begin with the worker nodes:

Copy these files securely to the worker nodes using scp utility

Root CA certificate – ca.pem
X509 Certificate for each worker node
Private Key of the certificate for each worker node

`for i in 0 1 2; do
  instance="${NAME}-worker-${i}"
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  scp -i ../ssh/${NAME}.id_rsa \
    ca.pem ${instance}-key.pem ${instance}.pem ubuntu@${external_ip}:~/; \
done`

[Copy SCP](./images/copy-scp.PNG)

Master or Controller node: – Note that only the api-server related files will be sent over to the master nodes:

`for i in 0 1 2; do
instance="${NAME}-master-${i}" \
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  scp -i ../ssh/${NAME}.id_rsa \
    ca.pem ca-key.pem service-account-key.pem service-account.pem \
    master-kubernetes.pem master-kubernetes-key.pem ubuntu@${external_ip}:~/;
done`

[Master or Controller node](./images/mast-cont-1.PNG)

[Master or Controller node](./images/mast-cont-2.PNG)

The kube-proxy, kube-controller-manager, kube-scheduler, and kubelet client certificates will be used to generate client authentication configuration files later.

STEP 5 USE `KUBECTL` TO GENERATE KUBERNETES CONFIGURATION FILES FOR AUTHENTICATION
All the work you are doing right now is ensuring that you do not face any difficulties by the time the Kubernetes cluster is up and running. In this step, you will create some files known as kubeconfig, which enables Kubernetes clients to locate and authenticate to the Kubernetes API Servers.

You will need a client tool called kubectl to do this. And, by the way, most of your time with Kubernetes will be spent using kubectl commands.

Now it’s time to generate kubeconfig files for the kubelet, controller manager, kube-proxy, and scheduler clients and then the admin user.

First, let us create a few environment variables for reuse by multiple commands.

`KUBERNETES_API_SERVER_ADDRESS=$(aws elbv2 describe-load-balancers --load-balancer-arns ${LOAD_BALANCER_ARN} --output text --query 'LoadBalancers[].DNSName')`

[KUBERNETES_API_SERVER_ADDRESS](./images/kuber-api-server.PNG)

1. Generate the kubelet kubeconfig file
For each of the nodes running the kubelet component, it is very important that the client certificate configured for that node is used to generate the kubeconfig. This is because each certificate has the node’s DNS name or IP Address configured at the time the certificate was generated. It will also ensure that the appropriate authorization is applied to that node through the Node Authorizer

Below command must be run in the directory where all the certificates were generated.

`for i in 0 1 2; do

instance="${NAME}-worker-${i}"
instance_hostname="ip-172-31-0-2${i}"

 # Set the kubernetes cluster in the kubeconfig file
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://$KUBERNETES_API_SERVER_ADDRESS:6443 \
    --kubeconfig=${instance}.kubeconfig

# Set the cluster credentials in the kubeconfig file
  kubectl config set-credentials system:node:${instance_hostname} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

# Set the context in the kubeconfig file
  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:node:${instance_hostname} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done`

[KUBERNETES_API_SERVER_ADDRESS](./images/kube-kubeconfig-file.PNG)

List the output

`ls -ltr *.kubeconfig`

[List Output](./images/ls-ltr-kubconf.PNG)

Open up the kubeconfig files generated and review the 3 different sections that have been configured:

Cluster
Credentials
And Kube Context
Kubeconfig file is used to organize information about clusters, users, namespaces and authentication mechanisms. By default, kubectl looks for a file named config in the $HOME/.kube directory. You can specify other kubeconfig files by setting the KUBECONFIG environment variable or by setting the --kubeconfig flag. To get to know more how to create your own kubeconfig files – read this documentation.

Context part of kubeconfig file defines three main parameters: cluster, namespace and user. You can save several different contexts with any convenient names and switch between them when needed.

`kubectl config use-context %context-name%`

2. Generate the kube-proxy kubeconfig:

`{
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_API_SERVER_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}`

[kube-proxy kubeconfig](./images/kube-proxy-kubeconfig.PNG)

3. Generate the Kube-Controller-Manager kubeconfig

Notice that the --server is set to use 127.0.0.1. This is because, this component runs on the API-Server so there is no point routing through the Load Balancer.

`{
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}`

[Kube-Controller-Manager](./images/kube-cont-mgr.PNG)

4. Generating the Kube-Scheduler Kubeconfig

`{
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}`

[Kube-Scheduler Kubeconfig](./images/kube-sche-kubconfg.PNG)

5. Finally, generate the kubeconfig file for the admin user:

`{
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_API_SERVER_ADDRESS}:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}`

[admin user](./images/admin-user.PNG)

TASK: Distribute the files to their respective servers, using scp and a for loop like we have done previously. This is a test to validate that you understand which component must go to which node.

STEP 6 PREPARE THE ETCD DATABASE FOR ENCRYPTION AT REST.
Step 6 Prepare the etcd database for encryption at rest.
Kubernetes uses etcd (A distributed key value store) to store variety of data which includes the cluster state, application configurations, and secrets. By default, the data that is being persisted to the disk is not encrypted. Any attacker that is able to gain access to this database can exploit the cluster since the data is stored in plain text. Hence, it is a security risk for Kubernetes that needs to be addressed.

To mitigate this risk, we must prepare to encrypt etcd at rest. "At rest" means data that is stored and persists on a disk. Anytime you hear "in-flight" or "in transit" refers to data that is being transferred over the network. "In-flight" encryption is done through TLS.

Generate the encryption key and encode it using base64:

`ETCD_ENCRYPTION_KEY=$(head -c 64 /dev/urandom | base64)`

See the output that will be generated when called. Yours will be a different random string.

`echo $ETCD_ENCRYPTION_KEY`

[admin user](./images/encry-key.PNG)

** Create an encryption-config.yaml file as documented officially by kubernetes:

`cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ETCD_ENCRYPTION_KEY}
      - identity: {}
EOF`

Send the encryption file to the Controller nodes using scp and a for loop.

Bootstrap etcd cluster
TIPS: Use a terminal multi-plexer like multi-tabbed putty or tmux to work with multiple terminal sessions simultaneously. It will make your life easier, especially when you need to work on multiple nodes and run the same command across all nodes. Imagine repeating the same commands on 10 different nodes, and you don not intend to start automating with a configuration management tool like Ansible yet.

The primary purpose of the etcd component is to store the state of the cluster. This is because Kubernetes itself is stateless. Therefore, all its stateful data will persist in etcd. Since Kubernetes is a distributed system – it needs a distributed storage to keep persistent data in it. etcd is a highly-available key value store that fits the purpose. All K8s cluster configurations are stored in a form of key value pairs in etcd, it also stores the actual and desired states of the cluster. etcd cluster is intelligent enough to watch for changes made on one instance and almost instantly replicate those changes to the rest of the instances, so all of them will be always reconciled.

NOTE: Don not just copy and paste the commands, ensure that you go through each and understand exactly what they will do on your servers. Use tools like tmux to make it easy to run commands on multiple terminal screens at once.

cd into the SSH folder before running the command

1. SSH into the controller server
 Master-1

`master_1_ip=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=${NAME}-master-0" \
--output text --query 'Reservations[].Instances[].PublicIpAddress')
ssh -i k8s-cluster-from-ground-up.id_rsa ubuntu@${master_1_ip}`

[Master-1](./images/master-1.PNG)

Master-2

`master_2_ip=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=${NAME}-master-1" \
--output text --query 'Reservations[].Instances[].PublicIpAddress')
ssh -i k8s-cluster-from-ground-up.id_rsa ubuntu@${master_2_ip}`

[Master-2](./images/master-2.PNG)

Master-3

`master_2_ip=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=${NAME}-master-1" \
--output text --query 'Reservations[].Instances[].PublicIpAddress')
ssh -i k8s-cluster-from-ground-up.id_rsa ubuntu@${master_2_ip}`

[Master-3](./images/master-3.PNG)

You should have a a similar pane like below. You should be able to see all the files that have been sent to the nodes.

2. Download and install etcd:

`wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.15/etcd-v3.4.15-linux-amd64.tar.gz"}`

[ETCD](./images/etcd-output.PNG)

3. Extract and install the etcd server and the etcdctl command line utility:

`{
tar -xvf etcd-v3.4.15-linux-amd64.tar.gz
sudo mv etcd-v3.4.15-linux-amd64/etcd* /usr/local/bin/
}`

[ETCD Server Command](./images/etcdctl-1.PNG)

[ETCD Server Command](./images/etcdctl-2.PNG)

4. Configure the etcd server:

`{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo chmod 700 /var/lib/etcd
  sudo cp ca.pem master-kubernetes-key.pem master-kubernetes.pem /etc/etcd/
}`

[ETCD Server Configure](./images/etcd-config.PNG)

5. The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers. Retrieve the internal IP address for the current compute instance:

`export INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)`

[Internal IP Address](./images/inter-add.PNG)

6. Each etcd member must have a unique name within an etcd cluster. Set the etcd name to node Private IP address so it will uniquely identify the machine:

`ETCD_NAME=$(curl -s http://169.254.169.254/latest/user-data/ \
  | tr "|" "\n" | grep "^name" | cut -d"=" -f2)

echo ${ETCD_NAME}`

[ETCD Name](./images/etcd-name.PNG)

7. Create the etcd.service systemd unit file:

The flags are well documented here:

`cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster master-0=https://172.31.0.10:2380,master-1=https://172.31.0.11:2380,master-2=https://172.31.0.12:2380 \\
  --cert-file=/etc/etcd/master-kubernetes.pem \\
  --key-file=/etc/etcd/master-kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/master-kubernetes.pem \\
  --peer-key-file=/etc/etcd/master-kubernetes-key.pem \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF`

[ETCD Service](./images/etcd-service.PNG)
